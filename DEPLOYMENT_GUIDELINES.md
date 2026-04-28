# PingFin26 – Deployment Guidelines
**Network Team | International Week 2026**

---

## Server Info

- **IP Address:** `192.168.137.232`
- **Docker Network:** `pingfin_net`

---

## Port Assignments

- **Clearing Bank (CB):** port `8080`
- **Bank A:** port `8081`
- **Bank B:** port `8082`

---

## How It Works

Each bank team creates their own **public GitHub repository** containing:
- Their application source code
- A `Dockerfile` at the root of the repo

The network team will clone your repo on the server, build the Docker image, and run your container. You do **not** need to deliver a `.tar` file.

---

## What We Need From Your Team

1. Link to your public GitHub repository
2. A working `Dockerfile` at the root of your repo
3. List of required environment variables (if any)
4. A health check endpoint at `/api/info/`
5. Name of your contact person

---

## Minimum Repo Structure

```
your-repo/
├── Dockerfile
└── ... (your application files)
```

---

## Dockerfile Example

```dockerfile
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 80
CMD ["python", "app.py"]
```

---

## Required API Endpoint

Your app must respond to:

```
GET /api/info/
```

Expected response:

```json
{
  "ok": true,
  "status": 200,
  "code": 2000,
  "message": "OK",
  "data": {
    "team": "Your Bank Name",
    "bic": "YOURBIC1",
    "members": ["name1", "name2", "name3"]
  }
}
```

---

## How We Deploy Your App

Once you send us your GitHub link, we run:

```bash
git clone https://github.com/yourteam/your-repo.git
cd your-repo
docker build -t teamname:latest .
docker run -d --name teamname_app --network pingfin_net -p PORT:80 teamname:latest
```

---

https://github.com/Muslim200/Projectweek
*Contact the network team if you have any questions.*
