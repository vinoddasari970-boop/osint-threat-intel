# Tailscale VPN Setup — Cross-Network Connection
## MongoDB (Windows) ↔ ELK Stack (Ubuntu)

## Overview

This document covers setting up Tailscale to create a secure private network between two machines on different networks:

- **Machine A** — MongoDB (Windows)
- **Machine B** — ELK Stack (Ubuntu)

Tailscale creates a peer-to-peer encrypted tunnel between machines using WireGuard under the hood. No port forwarding or firewall rules needed.

---

## Architecture

```
Machine A (Windows)          Machine B (Ubuntu)
MongoDB :27017      <——>     Elasticsearch :9200
Tailscale IP:                Tailscale IP:
100.x.x.x                   100.x.x.x
        |                         |
        +——— Tailscale Network ———+
           (Encrypted WireGuard)
```

---

## Prerequisites

- Admin access on both machines
- A free Tailscale account → [tailscale.com](https://tailscale.com)
- Internet access on both machines

---

## Step 1 — Create Tailscale Account

1. Go to [tailscale.com](https://tailscale.com)
2. Sign up using Google or GitHub (free tier is sufficient)
3. You will use this **same account** on both machines

---

## Step 2 — Install Tailscale

### Machine B — Ubuntu (ELK Stack)

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start and authenticate
sudo tailscale up
```

After running `tailscale up`, it prints a URL in the terminal.  
Open that URL in a browser and sign in with your Tailscale account.

### Machine A — Windows (MongoDB)

1. Download the Windows installer from [tailscale.com/download](https://tailscale.com/download)
2. Run the `.exe` installer
3. Sign in with the **same Tailscale account** used on Ubuntu

---

## Step 3 — Verify Both Machines Are Connected

Run on Ubuntu:

```bash
tailscale status
```

Expected output:

```
100.x.x.x    machine-a-name    windows    active
100.x.x.x    machine-b-name    ubuntu     active
```

Both machines must show `active` before proceeding.

---

## Step 4 — Test Connectivity

From Ubuntu, ping the Windows machine using its Tailscale IP:

```bash
ping <windows-tailscale-ip>
```

From Windows, ping Ubuntu using its Tailscale IP:

```powershell
ping <ubuntu-tailscale-ip>
```

Both pings should succeed with low latency.

---

## Step 5 — Configure MongoDB for Remote Access

By default MongoDB only listens on `localhost`. Change this to allow connections from Tailscale IP.

On **Machine A (Windows)**, find and edit `mongod.cfg`:

```
# Default location
C:\Program Files\MongoDB\Server\<version>\bin\mongod.cfg
```

Change the `bindIp` line:

```yaml
# Before
net:
  bindIp: 127.0.0.1

# After — add your Tailscale IP of Machine A
net:
  bindIp: 127.0.0.1,100.x.x.x
```

Restart MongoDB service on Windows:

```powershell
net stop MongoDB
net start MongoDB
```

---

## Step 6 — Test MongoDB Connection from Ubuntu

On **Machine B (Ubuntu)**:

```bash
# Install mongo shell if not present
sudo apt install mongodb-clients -y

# Connect to MongoDB on Windows via Tailscale IP
mongosh "mongodb://<windows-tailscale-ip>:27017"
```

If it connects — the tunnel is working correctly.

---

## Step 7 — Configure Elasticsearch for Remote Access

By default Elasticsearch also binds to `localhost` only.

On **Machine B (Ubuntu)**, edit:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Add or update these lines:

```yaml
network.host: 0.0.0.0
discovery.type: single-node
```

Restart Elasticsearch:

```bash
sudo systemctl restart elasticsearch
```

Verify it is running:

```bash
curl http://localhost:9200
```

---

## Step 8 — Test Full Connectivity

From **Machine A (Windows)**, open browser or PowerShell and hit:

```
http://<ubuntu-tailscale-ip>:9200
```

Expected response:

```json
{
  "name" : "...",
  "cluster_name" : "elasticsearch",
  "version" : { ... },
  "tagline" : "You Know, for Search"
}
```

---

## Connection Summary

| Service       | Machine  | OS      | Port  | Tailscale IP  |
|---------------|----------|---------|-------|---------------|
| MongoDB       | Machine A| Windows | 27017 | 100.x.x.x     |
| Elasticsearch | Machine B| Ubuntu  | 9200  | 100.x.x.x     |
| Kibana        | Machine B| Ubuntu  | 5601  | 100.x.x.x     |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `tailscale status` shows only one machine | Re-run `sudo tailscale up` on the missing machine and re-authenticate |
| MongoDB connection refused | Check `bindIp` in `mongod.cfg` and restart the service |
| Elasticsearch returns 403 or no response | Check `network.host` in `elasticsearch.yml` and restart |
| Ping fails between machines | Both machines must be logged into the **same** Tailscale account |

---

## Security Notes

- Tailscale traffic is encrypted end-to-end using WireGuard
- Only machines on your Tailscale account can access each other
- Do **not** expose MongoDB port `27017` to the public internet
- For production, add MongoDB authentication (`--auth` flag)

---

## Next Step

Once connectivity is confirmed, the Python pipeline script reads from MongoDB (`100.x.x.x:27017`) and writes to Elasticsearch (`100.x.x.x:9200`) to build the threat landscape dashboard in Kibana.
