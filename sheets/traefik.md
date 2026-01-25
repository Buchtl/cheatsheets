Traefik + ACME via Docker Compose is a solid setup 👍
Below is a **clean, production-ready `docker-compose.yml`** that:

* Runs **Traefik v2**
* Automatically gets **Let’s Encrypt certificates (ACME)**
* Uses **HTTP-01 challenge**
* Watches Docker for labels
* Exposes **:80 and :443**
* Stores certs safely

---

## `docker-compose.yml`

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    restart: unless-stopped

    ports:
      - "80:80"
      - "443:443"

    command:
      # Dashboard (disable in public environments or protect it)
      - "--api.dashboard=true"

      # Docker provider
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"

      # Entrypoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"

      # Redirect HTTP → HTTPS
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"

      # ACME / Let's Encrypt
      - "--certificatesresolvers.le.acme.email=you@example.com"
      - "--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.le.acme.httpchallenge=true"
      - "--certificatesresolvers.le.acme.httpchallenge.entrypoint=web"

    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./letsencrypt:/letsencrypt"

    labels:
      # Optional: expose dashboard via HTTPS
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.example.com`)"
      - "traefik.http.routers.traefik.entrypoints=websecure"
      - "traefik.http.routers.traefik.service=api@internal"
      - "traefik.http.routers.traefik.tls.certresolver=le"
```

---

## Important setup steps (don’t skip these)

### 1️⃣ Create the ACME storage file with correct permissions

Traefik **will fail silently** if this is wrong.

```bash
mkdir letsencrypt
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
```

---

### 2️⃣ DNS requirements

* `example.com` (and subdomains) **must point to this server**
* Ports **80 and 443 must be open**

---

### 3️⃣ Example app behind Traefik

```yaml
services:
  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=le"
```

Traefik will:

* Detect the container
* Request a cert automatically
* Serve HTTPS 🚀

---

## Common tweaks (tell me if you want these)

* 🔐 Protect the dashboard with auth
* 🌍 Use **DNS-01 challenge** (Cloudflare, Route53, etc.)
* 🧱 Run on **Docker Swarm**
* 🥧 Raspberry Pi–specific optimizations
* 📦 Split static vs dynamic config files

