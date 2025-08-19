
## Prometheus node-exporter
A clean, safe way to set it up so only **one specific IP** can access **port 9100** and everyone else is blocked. I’ll show the commands step by step.

---

### 1. Reset any previous rules on port 9100 (optional but safe)

```bash
sudo ufw delete allow 9100
sudo ufw delete deny 9100
```

---

### 2. Allow the specific IP

Replace `111.0.111.11` with your IP:

```bash
sudo ufw allow from 111.0.111.11 to any port 9100
```

---

### 3. Deny all other IPs

```bash
sudo ufw deny 9100
```

> UFW processes rules **top to bottom**, so the specific allow will take precedence over the deny.

---

### 4. Check the rules

```bash
sudo ufw status numbered
```

You should see:

```
To                         Action      From
--                         ------      ----
9100                       ALLOW       111.0.111.11
9100                       DENY        Anywhere
```

---

This guarantees only `111.0.111.11` can reach port 9100, everyone else is blocked.

---
