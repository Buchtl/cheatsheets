Yes. A **Step CA** container can act as an **ACME provider** using a specific root CA and intermediate CA, but there are a few requirements.

# How Step CA works

[Step CA official documentation](https://smallstep.com/docs/step-ca/?utm_source=chatgpt.com)

Step CA issues certificates from an internal PKI hierarchy:

```
Root CA (offline or protected)
    |
    +-- Intermediate CA (used by Step CA)
            |
            +-- End-entity certificates issued via ACME
```

The ACME endpoint serves certificates signed by the configured intermediate CA. Clients must trust the corresponding root CA.

### Using your own existing Root and Intermediate

This is supported.

When initializing or configuring Step CA, you can provide:

* Your existing root certificate (`root_ca.crt`)
* Your existing intermediate certificate (`intermediate_ca.crt`)
* The private key for the intermediate CA (`intermediate_ca_key`)
* Optionally keep the root private key offline

The Step CA instance only needs access to the intermediate private key for normal certificate issuance.

A typical configuration references:

```json
{
  "root": "/home/step/certs/root_ca.crt",
  "crt": "/home/step/certs/intermediate_ca.crt",
  "key": "/home/step/secrets/intermediate_ca_key"
}
```

### Enabling ACME

Step CA exposes an ACME provisioner:

```bash
step ca provisioner add acme \
  --type ACME
```

or via the configuration file.

Clients can then use an ACME directory URL similar to:

```text
https://ca.example.com/acme/acme/directory
```

and tools such as:

* [Certbot](https://certbot.eff.org/?utm_source=chatgpt.com)
* [Traefik](https://traefik.io/?utm_source=chatgpt.com)
* [Caddy](https://caddyserver.com/?utm_source=chatgpt.com)
* [NGINX Proxy Manager](https://nginxproxymanager.com/?utm_source=chatgpt.com)
* [lego ACME client](https://go-acme.github.io/lego/?utm_source=chatgpt.com)

can request certificates from your Step CA.

### Common deployment pattern

Many organizations deploy:

```
Offline Root CA
    |
    +-- Step CA Intermediate
            |
            +-- ACME endpoint
                    |
                    +-- Traefik
                    +-- Internal web servers
                    +-- Kubernetes ingress
```

This allows automatic certificate issuance while keeping the root key offline.

### Things to watch for

1. **The intermediate certificate must have CA capabilities**

   * `Basic Constraints: CA:TRUE`
   * Appropriate `Key Usage` values (`keyCertSign`, `cRLSign`).

2. **The intermediate must be signed by the root you provide.**

3. **Clients must trust the root certificate.**

   * Browsers and systems will not automatically trust your private root CA.
   * You must distribute the root certificate to all clients.

4. **You need the intermediate private key.**

   * Without it, Step CA cannot issue certificates.

5. **Certificate chain presentation.**

   * Step CA should present the issued certificate plus the intermediate chain to ACME clients.

If you're running Step CA in Docker and already have a root certificate, intermediate certificate, and intermediate private key, I can show the exact `docker-compose.yml` and `ca.json` configuration needed to import them and enable ACME.

# Config

Assuming you already have:

```text
certs/
├── root_ca.crt
├── intermediate_ca.crt
└── intermediate_ca_key
```

you can mount them into the container and configure Step CA to use the existing PKI hierarchy.

## Directory layout

```text
step-ca/
├── docker-compose.yml
├── config/
│   └── ca.json
├── certs/
│   ├── root_ca.crt
│   ├── intermediate_ca.crt
│   └── intermediate_ca_key
└── secrets/
    └── password.txt
```

`password.txt` contains the password protecting the intermediate private key.

---

## docker-compose.yml

```yaml
version: "3.9"

services:
  step-ca:
    image: smallstep/step-ca:latest
    container_name: step-ca
    restart: unless-stopped

    ports:
      - "9000:9000"

    volumes:
      - ./config:/home/step/config
      - ./certs:/home/step/certs:ro
      - ./secrets:/home/step/secrets:ro

    environment:
      DOCKER_STEPCA_INIT_NAME: "My Internal CA"

    command:
      - /home/step/config/ca.json

    healthcheck:
      test: ["CMD", "step-ca", "health"]
      interval: 30s
      timeout: 5s
      retries: 3
```

---

## ca.json

This is a minimal example using your existing certificates.

```json
{
  "root": "/home/step/certs/root_ca.crt",

  "crt": "/home/step/certs/intermediate_ca.crt",

  "key": "/home/step/certs/intermediate_ca_key",

  "address": ":9000",

  "dnsNames": [
    "ca.example.com"
  ],

  "logger": {
    "format": "text"
  },

  "db": {
    "type": "badgerv2",
    "dataSource": "/home/step/db"
  },

  "authority": {
    "provisioners": [
      {
        "type": "ACME",
        "name": "acme"
      }
    ]
  },

  "tls": {
    "cipherSuites": [
      "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256",
      "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
    ],
    "minVersion": 1.2,
    "maxVersion": 1.3
  }
}
```

---

## More realistic ACME setup

Most installations also configure ACME policies:

```json
{
  "type": "ACME",
  "name": "acme",
  "claims": {
    "maxTLSCertDuration": "2160h"
  }
}
```

This would allow certificates up to 90 days.

---

## ACME directory URL

After startup, your ACME endpoint is typically:

```text
https://ca.example.com/acme/acme/directory
```

You can verify it:

```bash
curl https://ca.example.com/acme/acme/directory
```

---

## Using Traefik

Example Traefik resolver:

```yaml
certificatesResolvers:
  stepca:
    acme:
      caServer: https://ca.example.com/acme/acme/directory
      email: admin@example.com
      storage: /etc/traefik/acme.json

      tlsChallenge: {}
```

You must also make the Traefik container trust your root CA.

---

## Important note about importing an existing CA

Step CA expects:

1. The certificate in `crt` to be a valid CA certificate.
2. The private key in `key` to match that certificate.
3. The CA certificate to chain up to the root in `root`.
4. The intermediate certificate to contain:

   * `CA:TRUE`
   * `keyCertSign`
   * `cRLSign`

You can verify that with:

```bash
openssl x509 -in intermediate_ca.crt -text -noout
```

Look for:

```text
X509v3 Basic Constraints:
    CA:TRUE

X509v3 Key Usage:
    Certificate Sign, CRL Sign
```

One thing to be aware of: recent Step CA versions usually generate a more complete `ca.json` containing fields such as `authorityConfig`, encrypted key handling, and JWK-based provisioners. If you're starting from scratch with an existing CA, I generally recommend running:

```bash
docker run --rm -it \
  -v $(pwd):/home/step \
  smallstep/step-ca \
  step ca init
```

and then replacing the generated root/intermediate certs with your own, rather than hand-writing the entire configuration. That avoids version-specific configuration issues.

# Testing

If ACME is enabled, you **cannot directly obtain a certificate with plain `curl`**, because ACME is a multi-step protocol (account registration, challenge validation, order finalization, certificate retrieval).

However, you can use `curl` to verify that the ACME endpoint is working.

### 1. Check the ACME directory

```bash
curl -k https://ca.example.com/acme/acme/directory
```

Expected output:

```json
{
  "newNonce": "...",
  "newAccount": "...",
  "newOrder": "...",
  "revokeCert": "...",
  "keyChange": "..."
}
```

If you get this JSON, the ACME provisioner is active.

---

### 2. Request a certificate using an ACME client

The easiest test is with `step` itself:

```bash
step ca certificate \
  test.example.com \
  test.crt \
  test.key \
  --provisioner acme \
  --acme https://ca.example.com/acme/acme/directory
```

Or using `lego`:

```bash
lego \
  --server https://ca.example.com/acme/acme/directory \
  --email admin@example.com \
  --dns your-dns-provider \
  --domains test.example.com \
  run
```

---

### 3. Fastest end-to-end test with OpenSSL + ACME client

Install a lightweight ACME client such as `acme.sh`:

```bash
curl https://get.acme.sh | sh
```

Then:

```bash
acme.sh \
  --server https://ca.example.com/acme/acme/directory \
  --issue \
  -d test.example.com \
  --standalone
```

This exercises the entire ACME flow.

---

### 4. If you only want to test Step CA issuance (not ACME)

Step CA can issue certificates directly through a provisioner:

```bash
step ca certificate \
  test.example.com \
  test.crt \
  test.key
```

This bypasses ACME entirely and confirms that:

* the imported root certificate is valid,
* the imported intermediate certificate is valid,
* the intermediate private key works,
* Step CA can sign certificates.

---

### 5. Verify the issued certificate

After obtaining a certificate:

```bash
openssl verify \
  -CAfile root_ca.crt \
  -untrusted intermediate_ca.crt \
  test.crt
```

Expected:

```text
test.crt: OK
```

For a complete ACME test, I usually recommend **`acme.sh`** or **`lego`** because they exercise exactly what Traefik, Caddy, NGINX Proxy Manager, and other ACME clients will do against your Step CA server.
