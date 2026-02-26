# Coolify deployment plan – Mail server (Mailu)

## 1. Is the current directory structure a good choice?

**Yes.** Keeping everything under `mailu/` is a good choice because:

| Aspect | Why it works |
|--------|----------------|
| **Single compose root** | Coolify runs `docker compose` from the directory that contains the compose file. Having `docker-compose.yml` and `mailu.env` in the same folder means `env_file: mailu.env` works without path hacks. |
| **One service in Coolify** | One Coolify “service” = one compose stack. Your whole Mailu stack (front, admin, imap, smtp, etc.) is one logical service, so one directory = one compose file = correct. |
| **Repo layout** | If you add more Coolify services later (e.g. another app), you’d add another folder with its own compose file and point Coolify at that folder. So `mail-server/` = repo, `mail-server/mailu/` = this mail service. |

**Recommendation:** Keep the structure as-is. In Coolify, set the **compose file path** (or “source”) to the `mailu` directory (or the repo root with compose path `mailu/docker-compose.yml` if Coolify supports that). The important part is that the **working directory** when compose runs contains both `docker-compose.yml` and `mailu.env`.

---

## 2. Coolify workflow (project → service → compose)

```
Coolify Project (e.g. "Mail")
    └── Service (e.g. "Mailu")
            └── Source: Git repo or upload
            └── Compose file: mailu/docker-compose.yml  (or root = mailu folder)
            └── Env / build settings as needed
```

- **One project** for “mail” keeps things grouped.
- **One service** for Mailu, deployed from the single compose file in `mailu/`.
- No need to split Mailu into multiple Coolify services; the compose file already defines all containers (front, resolver, admin, imap, smtp, antispam, webmail, redis).

---

## 3. Port strategy – avoid conflicts

### Ports exposed by Mailu (all on the `front` container)

| Port | Protocol | Purpose |
|------|----------|---------|
| 80 | HTTP | Web (redirect to HTTPS), ACME (Let’s Encrypt) |
| 443 | HTTPS | Admin, webmail, API |
| 25 | SMTP | Inbound mail |
| 465 | SMTPS | Submission (encrypted) |
| 587 | Submission | Submission (STARTTLS) |
| 110 | POP3 | Insecure POP3 (optional to close) |
| 995 | POP3S | POP3 over TLS |
| 143 | IMAP | Insecure IMAP (optional to close) |
| 993 | IMAPS | IMAP over TLS |
| 4190 | Sieve | Manage filters |

### How to avoid port conflicts

1. **Single mail server on the host**  
   If this Coolify server runs **only** this Mailu stack (and maybe other apps behind a reverse proxy that don’t use 25/465/587/143/993/995), then binding to `0.0.0.0:80:80`, etc., is correct. No other service should use these ports.

2. **Coolify’s own proxy (Traefik/Caddy)**  
   - Coolify often uses a shared reverse proxy. If that proxy binds to **80** and **443** on the host, you must not also publish 80/443 from Mailu on the same host, or you get a conflict.  
   - **Option A – Recommended for mail:** Deploy this stack on a **dedicated server** (or dedicated public IP) where Mailu’s `front` is the only thing using 80, 443, 25, 465, 587, 110, 995, 143, 993, 4190. No Traefik/Caddy on those ports.  
   - **Option B – Same host as Coolify proxy:** Don’t publish 80/443 from the compose file; expose only 25, 465, 587, 110, 995, 143, 993, 4190. Put admin/webmail behind Coolify’s proxy via a separate internal port (e.g. expose 8080 from `front` and proxy `mail.swiftloopmarketing.com` to it). This requires Mailu config (and possibly custom nginx) so admin/webmail work over the proxy; it’s more complex and not the default Mailu setup.

3. **Multiple compose stacks on the same host**  
   - Each stack that publishes host ports must use **different host ports**.  
   - Mailu needs the standard ports (25, 465, 587, …) for compatibility (MX, clients). So on a given host, only **one** stack should use 25, 465, 587, 143, 993, 110, 995.  
   - If you run another stack that uses 80/443, either use another host or don’t publish 80/443 from Mailu and use Option B above.

**Practical checklist:**

- [ ] Decide: is this host (or IP) **only** for mail, or shared with a web proxy?
- [ ] If **mail-only**: keep current `ports` in compose; ensure no other service uses 80, 443, 25, 465, 587, 110, 995, 143, 993, 4190.
- [ ] If **shared**: remove 80/443 from compose and route admin/webmail via Coolify’s proxy (advanced).
- [ ] Run once: `netstat -tuln` or `ss -tuln` on the host and confirm nothing else is listening on 25, 587, 465, 143, 993, 80, 443 before deploy.

---

## 4. Volumes and paths

Your compose uses **absolute host paths** under `/mailu/`:

- `/mailu/certs`
- `/mailu/overrides/nginx`
- `/mailu/data`
- `/mailu/dkim`
- `/mailu/mail`
- `/mailu/mailqueue`
- `/mailu/filter`
- `/mailu/webmail`
- `/mailu/redis`
- `/mailu/overrides/dovecot`
- `/mailu/overrides/postfix`
- `/mailu/overrides/rspamd`
- `/mailu/overrides/roundcube`

On Coolify:

- If the server is Linux and you’re okay with a fixed path, create `/mailu/` (and subdirs) on the host and leave the compose as-is; Coolify will use the host’s filesystem.
- If Coolify assigns a project/service-specific path (e.g. `/data/coolify/services/<id>/`), you’d need to replace `/mailu/` in the compose with that path or use Compose env vars (e.g. `VOLUME_BASE`) and set them in Coolify. So: **check how Coolify mounts volumes** (bind mount from host path vs. project path). If it uses a dynamic path, we can switch the compose to something like `${VOLUME_BASE:-/mailu}/certs`, etc., and set `VOLUME_BASE` in Coolify.

---

## 5. Network subnet

Compose defines:

```yaml
networks:
  default:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.203.0/24
```

- This is the **internal** bridge for this stack only. Other stacks get their own bridge, so 192.168.203.0/24 won’t conflict with other compose projects.
- Only change it if 192.168.203.0/24 is already used on the host for something else (uncommon). Default is fine.

---

## 6. Pre-deploy checklist

- [ ] **Ports:** Nothing else on host listening on 25, 465, 587, 80, 443, 143, 993, 110, 995, 4190 (or plan proxy approach).
- [ ] **Coolify source:** Repo or upload points to this repo; compose path = `mailu/docker-compose.yml` or context = `mailu/` so `mailu.env` is next to the compose file.
- [ ] **Secrets:** Replace `SECRET_KEY`, `INITIAL_ADMIN_PW`, and any other secrets in `mailu.env` (use Coolify env vars or a secret manager if available).
- [ ] **Domain/DNS:** `mail.swiftloopmarketing.com` (and MX for `swiftloopmarketing.com`) points to this server’s IP.
- [ ] **Volumes:** Either create `/mailu/...` on host or adapt compose to Coolify’s volume path and set env accordingly.
- [ ] **TLS:** `TLS_FLAVOR=letsencrypt` is set; ensure 80 (or 443) is reachable for ACME if you use Let’s Encrypt.

---

## 7. Summary

| Question | Answer |
|----------|--------|
| Is the directory a good choice? | **Yes.** One folder `mailu/` with one compose + one env = one Coolify service, clean and correct. |
| Port conflicts? | Avoid by: one mail stack per host on standard ports, or no 80/443 publish and proxy admin/webmail. Check with `ss -tuln` before deploy. |
| Compose in Coolify? | One project → one service → one compose file (`mailu/docker-compose.yml`), run from a context where `mailu.env` is present. |
| Volumes? | Use current `/mailu/` paths if Coolify runs on a host where you control paths; otherwise parameterize with env (e.g. `VOLUME_BASE`) and set in Coolify. |

If you tell me whether this Coolify instance is “mail-only” or “shared with a web proxy,” I can suggest an exact `ports:` block and any small compose/env changes (e.g. volume base path) for that case.

---

## 8. Safe Coolify deployment (ports configured)

To deploy Mailu on the **same host** as Coolify's proxy without port conflicts, use the Coolify override so only mail ports are published; Coolify's proxy keeps 80/443 and routes web traffic to Mailu's `front` container internally.

### What's in place

| File | Purpose |
|------|--------|
| `mailu/docker-compose.coolify.yml` | Override that **removes 80 and 443** from the `front` container and **keeps** 25, 465, 587, 110, 995, 143, 993, 4190. No port conflict with Coolify. |
| `mailu/mailu.env` | Optional `REAL_IP_HEADER` / `REAL_IP_FROM` (commented) for when web traffic goes through Coolify's proxy. |

### Deploy steps on Coolify

1. **Compose files** – In the Coolify service, set compose file(s) so the override is used: e.g. `docker-compose.yml,docker-compose.coolify.yml`. If Coolify only allows one file, merge the override's `front.ports` into `docker-compose.yml` (no 80/443).
2. **Route web to Mailu** – Add domain `mail.swiftloopmarketing.com` and set target to `front` container, port **80** (internal). Coolify routes 80/443 to `front:80`.
3. **Let's Encrypt** – Coolify can issue the cert for the web UI. For mail (IMAP/SMTP) TLS either: (A) ensure Coolify forwards `/.well-known/acme-challenge/` to `front` so Mailu can use HTTP-01 and keep `TLS_FLAVOR=letsencrypt`, or (B) use Coolify's cert and mount into `/mailu/certs` with `TLS_FLAVOR=cert`.
4. **Real client IP** – Uncomment `REAL_IP_HEADER` and `REAL_IP_FROM` in `mailu.env` (e.g. `REAL_IP_FROM=172.16.0.0/12`).
5. **Mail ports** – 25, 465, 587, 110, 995, 143, 993, 4190 stay published from this stack; no conflict with Coolify.

### Ports when using the Coolify override

| Port | Published by Mailu? | Handled by |
|------|--------------------|------------|
| 80, 443 | No | Coolify proxy to `front:80` |
| 25, 465, 587, 110, 995, 143, 993, 4190 | Yes | This stack |
