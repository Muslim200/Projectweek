# PingFin26 – Network Team Setup Documentation
**International Week 2026 | Team Netwerk**

---

## 1. Hardware Overview

| Component | Model |
|-----------|-------|
| Server | Dell OptiPlex-7040 |
| Switch | HP V1910 24G (JE006A) |
| OS | Ubuntu (pre-installed) |

**Network topology:**
```
[Laptop X] ──── (WiFi) ──── Internet
[Laptop X] ──┐
[Laptop 2] ──┤──── [HP Switch] ──── [Server 192.168.137.232]
[Laptop 3] ──┘
```

Internet comes from the school WiFi via Laptop X using Windows ICS (Internet Connection Sharing). The server and all laptops are connected via ethernet cables through the switch.

---

## 2. Server Configuration

### IP Address
- **Server IP:** `192.168.137.232`
- The server receives its IP via Windows ICS (DHCP from Laptop X)
- Static IP configured via Ubuntu Network Manager GUI to prevent IP changes after reboot

### Network Settings (static)
- Address: `192.168.137.232`
- Netmask: `255.255.255.0`
- Gateway: `192.168.137.1` (Laptop X)
- DNS: `8.8.8.8`

### Docker Installation
```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

### Docker Network
```bash
docker network create pingfin_net
```

All containers must be connected to `pingfin_net` to communicate with each other.

---

## 3. Port Assignments

| Service | Container Name | Port |
|---------|---------------|------|
| Clearing Bank (CB) | cb_app | 8080 |
| Bank A | banka_app | 8081 |
| Bank B | bankb_app | 8082 |

---

## 4. Deployment Flow

When a bank team provides their GitHub repository link:

```bash
# Clone the repo
git clone https://github.com/bankteam/their-repo.git
cd their-repo

# Build the Docker image
docker build -t teamname:latest .

# Run the container on pingfin_net
docker run -d --name teamname_app --network pingfin_net -p PORT:80 teamname:latest

# Verify it works
curl http://192.168.137.232:PORT/api/info/
```

---

## 5. Inter-Container Communication

Containers on `pingfin_net` can reach each other using the **container name** as hostname. No IP address needed.

Example: `caller_app` calling `dummy_app`:
```python
requests.get('http://dummy_app/api/info/')
```

This works because Docker provides automatic DNS resolution within the same network.

**Verified with:**
```bash
curl http://192.168.137.232:8081/api/call/
# Response:
# {"ok": true, "message": "Successfully called dummy_app", "dummy_app_response": {...}}
```

---

## 6. Test Containers (Day 2)

Two test containers were built to validate the infrastructure before real bank apps were available.

### dummy_api (port 8080)
Simple Flask API exposing mock transaction data.

**app.py:**
```python
from flask import Flask, jsonify

app = Flask(__name__)

TRANSACTIONS = [
    {"id": 1, "from": "ARSPBE22", "to": "BNKABE99", "amount": 150.00, "currency": "EUR", "status": "completed"},
    {"id": 2, "from": "BNKABE99", "to": "ARSPBE22", "amount": 320.50, "currency": "EUR", "status": "completed"},
    {"id": 3, "from": "ARSPBE22", "to": "BNKCBE88", "amount": 75.00, "currency": "EUR", "status": "pending"},
]

@app.route('/api/info/')
def info():
    return jsonify({"ok": True, "status": 200, "code": 2000, "message": "OK", "data": []})

@app.route('/api/transactions/')
def transactions():
    return jsonify({"ok": True, "status": 200, "code": 2000, "message": "OK", "data": TRANSACTIONS})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

### caller_api (port 8081)
Flask API that calls dummy_api internally to prove inter-container communication.

**app.py:**
```python
import requests
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/api/info/')
def info():
    return jsonify({"ok": True, "status": 200, "code": 2000, "message": "OK", "data": []})

@app.route('/api/call/')
def call_other():
    try:
        response = requests.get('http://dummy_app/api/transactions/', timeout=5)
        return jsonify({"ok": True, "message": "Successfully called dummy_app", "dummy_app_response": response.json()})
    except Exception as e:
        return jsonify({"ok": False, "message": str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

**Build & run commands:**
```bash
# dummy_api
cd ~/dummy_api
docker build -t dummy_api:latest .
docker run -d --name dummy_app --network pingfin_net -p 8080:80 dummy_api:latest

# caller_api
cd ~/caller_api
docker build -t caller_api:latest .
docker run -d --name caller_app --network pingfin_net -p 8081:80 caller_api:latest
```

---

## 7. Problems & Solutions

### Problem 1 — Server stuck in initramfs on boot
**Symptom:** Server showed `(initramfs)` prompt and would not boot.
**Attempted fix:** `fsck -y /dev/sda1` — reported "clean" but did not resolve the issue.
**Solution:** Restarted the server multiple times; eventually booted normally. Server had a pre-existing OS issue.

---

### Problem 2 — USB boot failed ("Selected boot device failed")
**Symptom:** Tried to reinstall Ubuntu via USB but got "Selected boot device failed".
**Cause:** The teacher-provided USB was not correctly flashed.
**Solution:** Decided not to reinstall — the server eventually booted into the existing Ubuntu installation.

---

### Problem 3 — Server has no WiFi card
**Symptom:** Server could not connect to the internet wirelessly.
**Solution:** Used Windows ICS (Internet Connection Sharing) on Laptop X. Laptop X shares its WiFi connection via ethernet cable through the switch to the server.

---

### Problem 4 — Static IP conflicting with netplan
**Symptom:** Setting a static IP via Ubuntu GUI did not apply correctly because a netplan config file was also present and overriding the settings.
**Cause:** Both `/etc/netplan/01-network-manager-all.yaml` and the GUI were trying to manage the same interface.
**Solution:** Edited the netplan file directly to set the static IP `192.168.137.232`, then ran `sudo netplan apply`.

---

### Problem 5 — DNS not working after static IP setup
**Symptom:** `ping 8.8.8.8` worked but `ping google.com` failed. `apt update` also failed.
**Cause:** `/etc/resolv.conf` was pointing to `127.0.0.53` (systemd-resolved) which was not functioning correctly.
**Solution:**
```bash
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

---

### Problem 6 — Gateway unreachable (192.168.137.1)
**Symptom:** After configuring static IP, the server could not reach the gateway.
**Cause:** ICS on Laptop X had reset after a reboot, so the ethernet adapter no longer had IP `192.168.137.1`.
**Solution:** Re-enabled ICS on Laptop X (Network Settings → WiFi adapter → Properties → Sharing → enable for ethernet adapter).

---

### Problem 7 — `curl` not available inside container
**Symptom:** `docker exec caller_app curl http://dummy_app/api/info/` failed with "executable file not found".
**Cause:** `python:3.11-slim` is a minimal image without extra tools like `curl`.
**Solution:** Used Python's built-in `urllib` for testing:
```bash
docker exec caller_app python -c "import urllib.request; print(urllib.request.urlopen('http://dummy_app/api/info/').read())"
```

---

### Problem 8 — caller_api missing `/api/info/` endpoint
**Symptom:** `curl http://192.168.137.232:8081/api/info/` returned 404.
**Cause:** The `caller_api` was built without the `/api/info/` health check endpoint, even though the deployment guidelines required it for all containers.
**Solution:** Added the `/api/info/` route to `caller_api/app.py`, rebuilt and redeployed the container.

---

### Problem 9 — curl failed immediately after `docker run`
**Symptom:** Running `curl http://192.168.137.232:8081/api/info/` right after `docker run` returned "Failed to connect".
**Cause:** The container was not yet fully started when the curl command ran (timing issue).
**Solution:** Waited a few seconds and retried — the container was then reachable.

---

## 8. Useful Commands Reference

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# View container logs
docker logs container_name

# Stop and remove a container
docker stop container_name && docker rm container_name

# Check which network a container is on
docker inspect container_name | grep -A5 '"Networks"'

# Connect container to a network
docker network connect pingfin_net container_name

# Test inter-container communication
docker exec container_name python -c "import urllib.request; print(urllib.request.urlopen('http://other_container/api/info/').read())"

# Test from host
curl http://192.168.137.232:PORT/api/info/
```

---

## 9. GitHub Repository

Guidelines and documentation: https://github.com/Muslim200/Projectweek

---

*Network Team – PingFin26 – International Week 2026*
