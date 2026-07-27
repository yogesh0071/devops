# Real-Time WebSocket Chat Application

## Project Overview

A containerized real-time chat application deployed using Docker, Nginx reverse proxy, and automated CI/CD with GitHub Actions. This project demonstrates production-grade deployment practices including container networking, reverse proxy configuration, and continuous deployment automation.

---

## Live Application

**URL:** `http://50.17.47.45`

**Status:** Deployed and running with automated CI/CD

---

## Architecture Diagram
┌─────────────────────────────────────────────────────────────┐
│ User Browser │
│ (http://50.17.47.45) │
└──────────────────────────┬──────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ AWS EC2 Instance │
│ (50.17.47.45:80) │
├─────────────────────────────────────────────────────────────┤
│ │
│ ┌──────────────────────────────────────────────────────┐ │
│ │ NGINX Container (Reverse Proxy) │ │
│ │ │ │
│ │ • Serves frontend (index.html) │ │
│ │ • Routes /ws to backend │ │
│ │ • Preserves WebSocket upgrade headers │ │
│ │ • Listens on 0.0.0.0:80 │ │
│ └──────────────────┬───────────────────────────────────┘ │
│ │ (Docker Network) │
│ │ http://backend:8000 │
│ ▼ │
│ ┌──────────────────────────────────────────────────────┐ │
│ │ Backend Container (FastAPI + WebSocket) │ │
│ │ │ │
│ │ • Runs uvicorn on 0.0.0.0:8000 │ │
│ │ • Handles WebSocket connections │ │
│ │ • Broadcasts messages to all clients │ │
│ │ • Listens on Docker network only │ │
│ └──────────────────────────────────────────────────────┘ │
│ │
└─────────────────────────────────────────────────────────────┘
▲
│
CI/CD Automation
(GitHub Actions)
---

## What Was Broken (Issues Found and Fixed)

### Issue 1: Dockerfile - Backend Not Accepting Network Traffic

**Problem:** 
Backend was binding to `127.0.0.1` (localhost only), rejecting connections from other containers.

**Original Code (Line 13):**
```dockerfile
CMD ["uvicorn", "main:app", "--host", "127.0.0.1", "--port", "8000"]
```

**Why It Failed:**
- Nginx container tried to connect to backend via Docker network at `http://backend:8000`
- Backend rejected this because it only listened on 127.0.0.1 (same machine only)
- Connection refused error
- Users couldn't access the backend

**Fixed Code:**
```dockerfile
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Explanation:**
- `0.0.0.0` means "listen on all network interfaces"
- Includes Docker network bridge, host machine, and external connections
- Standard practice for containerized applications

---

### Issue 2: docker-compose.yml - Frontend Not Being Served

**Problem:**
Frontend volume mapping was commented out. Nginx had no HTML/CSS/JS files to serve.

**Original Code (Lines 17-18):**
```yaml
volumes:
  # TODO: Candidates need to map the frontend directory correctly to
  # - ./frontend:/usr/share/nginx/html:ro
  - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

**Why It Failed:**
- Nginx container had empty `/usr/share/nginx/html` directory
- Users browsed to `http://50.17.47.45` and got 404 Not Found
- Frontend never loaded

**Fixed Code:**
```yaml
volumes:
  - ./frontend:/usr/share/nginx/html:ro
  - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

**Explanation:**
- `:ro` means read-only (frontend shouldn't be modified at runtime)
- Maps local `./frontend` directory to Nginx's default HTML directory
- Nginx now serves index.html and all frontend assets

---

### Issue 3: nginx.conf - WebSocket Upgrade Headers Missing

**Problem:**
WebSocket upgrade headers were commented out. The reverse proxy wasn't telling the backend that a WebSocket connection should be established.

**Original Code (Lines 22-23):**
```nginx
# proxy_set_header Upgrade $http_upgrade;
# proxy_set_header Connection "upgrade";
```

**Why It Failed:**
- Browser sent WebSocket upgrade request with proper headers
- Nginx stripped these headers when proxying to backend
- Backend received normal HTTP request instead of WebSocket upgrade
- Connection attempt succeeded (200 OK) but WebSocket never established
- Real-time chat appeared to work but didn't actually work
- Users typed messages that never appeared in other tabs

**Fixed Code:**
```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

**Explanation:**
- `Upgrade: websocket` tells backend this is a WebSocket connection
- `Connection: upgrade` tells proxy to keep the connection open
- These are critical HTTP headers for WebSocket protocol
- Without them, WebSocket handshake fails silently

---

### Issue 4: nginx.conf - Wrong Backend Address

**Problem:**
Nginx was trying to connect to `localhost:8000` instead of the backend container.

**Original Code (Line 20):**
```nginx
proxy_pass http://localhost:8000/ws;
```

**Why It Failed:**
- `localhost` refers to the Nginx container itself
- Backend container has different hostname (`backend` or `chat-backend`)
- Nginx tried to connect to itself, got connection refused
- Users got 502 Bad Gateway or connection timeout

**Fixed Code:**
```nginx
proxy_pass http://backend:8000/ws;
```

**Explanation:**
- `backend` is the Docker service name from docker-compose.yml
- Docker's internal DNS resolves `backend` to the backend container's IP
- Works because both containers are on the same Docker network
- This is the standard way to reference other containers

---

## How Docker Containers Communicate

### Container Networking

**Docker Network Bridge:**
- Both Nginx and Backend containers share a network called `devops_default`
- Automatically created by docker-compose
- Internal DNS allows containers to find each other by service name
- Completely isolated from host machine's network

**Service Discovery:**
- When Nginx needs backend, it uses hostname `backend` (service name)
- Docker DNS resolves `backend` → backend container's internal IP
- This happens automatically, no manual configuration needed

**Network Isolation:**
Host Machine (EC2 Instance)
↓
└─ Docker Network: devops_default
├─ Nginx Container (internal IP)
└─ Backend Container (internal IP)
Only Nginx port 80 is exposed to host. Backend port 8000 stays internal.

### Port Mapping

**Nginx Port Mapping:**
```yaml
ports:
  - "80:80"  # Host:Container
```
- Host port 80 receives traffic from internet
- Mapped to container port 80
- Public IP 50.17.47.45:80 → Nginx container:80

**Backend Port Mapping:**
```yaml
expose:
  - "8000"
```
- No host port mapping (no `ports:` section)
- Only accessible within Docker network
- Security: Backend not directly exposed to internet

**Traffic Flow:**
Browser (Public Internet)
↓
Host Port 80 (50.17.47.45:80)
↓
Nginx Container Port 80
↓
Nginx Container Network Interface
↓
Docker Network Bridge
↓
Backend Container Port 8000
↓
FastAPI Application
---

## How Nginx Reverse Proxy Works

### What is a Reverse Proxy?

A reverse proxy sits between clients and servers, forwarding requests and responses. Clients think they're talking to one server, but requests are actually forwarded to different backends.

**Benefits:**
- Single entry point (one IP address: 50.17.47.45)
- Load distribution
- Security (backend hidden)
- SSL/TLS termination
- Request routing

### Frontend Requests

**Request:** Browser asks for `http://50.17.47.45/`

**Flow:**
Browser → Nginx:80 (GET /)
↓
Nginx looks up "/" in config
↓
Nginx serves file from /usr/share/nginx/html/index.html
↓
Browser receives index.html
**Nginx Configuration:**
```nginx
location / {
    root /usr/share/nginx/html;
    index index.html;
    try_files $uri $uri/ /index.html;
}
```

- Serves static files from the mounted frontend directory
- `try_files` supports single-page applications

### WebSocket Requests

**Request:** Browser opens WebSocket to `ws://50.17.47.45/ws`

**Flow:**
Browser → Nginx:80 (WebSocket Upgrade)
↓
Nginx receives HTTP Upgrade headers
↓
Nginx forwards to Backend:8000
↓
Backend receives Upgrade headers
↓
Backend establishes WebSocket connection
↓
WebSocket tunnel between Browser and Backend (through Nginx)
**Nginx Configuration:**
```nginx
location /ws {
    proxy_pass http://backend:8000/ws;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 86400s;
}
```

### Key Nginx Headers Explained

| Header | Purpose |
|--------|---------|
| `Upgrade: websocket` | Tells backend this is a WebSocket upgrade |
| `Connection: upgrade` | Tells Nginx to keep connection open |
| `Host` | Original host the browser requested |
| `X-Real-IP` | Real IP of the client (not the proxy) |
| `X-Forwarded-For` | Chain of IPs the request passed through |
| `X-Forwarded-Proto` | Original protocol (http vs https) |

---

## How WebSocket Works Through Nginx

### WebSocket Protocol Basics

WebSocket is a protocol for real-time, bidirectional communication over a single TCP connection.

**HTTP to WebSocket Upgrade Process:**

1. **Browser sends HTTP Upgrade request:**
```http
GET /ws HTTP/1.1
Host: 50.17.47.45
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

2. **Server accepts upgrade:**
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

3. **WebSocket connection established** (persistent)

### Through Nginx Reverse Proxy

**Without Nginx (Direct Connection):**
Browser → Backend:8000 (WebSocket)
**With Nginx (Production Setup):**
Browser → Nginx:80 (HTTP Upgrade)
↓
Nginx sees Upgrade headers
↓
Nginx forwards request to Backend:8000
↓
Backend receives Upgrade request
↓
Backend sends back 101 Switching Protocols
↓
Nginx tunnels all subsequent WebSocket frames
### Critical Requirements for WebSocket Through Proxy

**These headers MUST be present and forwarded:**
```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

**If these are missing:**
- Browser gets 200 OK response
- Browser thinks connection succeeded
- Nginx closes connection immediately after response
- WebSocket never actually connects
- Messages don't flow through

**This is why the commented headers were a critical bug.**

### Why the Long Timeout?

```nginx
proxy_read_timeout 86400s;  # 24 hours
```

- WebSocket connections are long-lived
- Default timeout is 60 seconds
- Without long timeout, Nginx would close idle WebSocket connections
- Users would disconnect after 1 minute of no messages
- 86400 seconds = 24 hours (prevents accidental disconnects)

---

## How CI/CD Pipeline Works

### GitHub Actions Workflow Overview

GitHub Actions runs automated tasks when code is pushed to the repository.

**Our workflow:**
1. Code pushed to GitHub
2. GitHub Actions automatically runs `deploy.yml`
3. Connects to EC2 instance (50.17.47.45) via SSH
4. Pulls latest code
5. Rebuilds Docker containers
6. Restarts application
7. No manual intervention needed

### Workflow File: `.github/workflows/deploy.yml`

```yaml
name: Deploy to EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Deploy to EC2
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: 22
          script: |
            cd ~/devops
            git pull origin main
            docker-compose down
            docker-compose up -d --build
```

### Workflow Execution Steps

**Step 1: Checkout Code**
```yaml
uses: actions/checkout@v3
```
- Downloads the repository code to GitHub's runner machine

**Step 2: Deploy to EC2**
```yaml
uses: appleboy/ssh-action@master
```
- Uses SSH to connect to EC2 at 50.17.47.45
- Executes deployment script on remote server

### GitHub Secrets (Security)

Sensitive information stored securely in GitHub:

**SSH_PRIVATE_KEY**
- EC2 private key for authentication
- Never exposed in logs
- Required to SSH into server

**SSH_HOST**
- Public IP address of EC2 instance
- Value: `50.17.47.45`

**SSH_USER**
- Username on EC2 instance
- Value: `ubuntu`

**Why Secrets Are Important:**
- Credentials never appear in workflow files (visible to public)
- Credentials never appear in build logs
- Only accessible to authorized deployments
- GitHub encrypts them at rest

### Deployment Script Breakdown

```bash
cd ~/devops
```
- Navigate to project directory on EC2

```bash
git pull origin main
```
- Download latest code from GitHub
- Fails if there are local changes (prevents overwriting)

```bash
docker-compose down
```
- Stop and remove all running containers
- Preserves volumes (no data loss)
- Prepares for new build

```bash
docker-compose up -d --build
```
- `-d` = detached mode (runs in background)
- `--build` = rebuild images with latest code
- Starts Nginx and Backend containers

### Deployment Workflow Timeline
Developer commits and pushes code
↓ (within 30 seconds)
GitHub Actions triggered
↓
Checkout code (2-3 seconds)
↓
SSH to EC2 50.17.47.45 (3-5 seconds)
↓
Pull latest code (2-3 seconds)
↓
Stop old containers (5 seconds)
↓
Rebuild images (1-2 minutes, depends on changes)
↓
Start new containers (5-10 seconds)
↓
Application live at http://50.17.47.45 (total: 1.5-3 minutes)
---

## Deployment Steps

### Local Development Setup

**Prerequisites:**
- Git installed
- Docker installed
- Docker Compose installed

**Clone Repository:**
```bash
git clone https://github.com/YOUR_USERNAME/devops.git
cd devops
```

**Start Containers Locally:**
```bash
docker-compose up -d --build
```

**Test Application:**
Open browser: http://localhost
**View Logs:**
```bash
docker-compose logs -f
```

**Stop Containers:**
```bash
docker-compose down
```

### Cloud Deployment on AWS (Step-by-Step)

#### Step 1: Launch EC2 Instance

1. Go to AWS Console → EC2 → Launch Instances
2. Configure:
   - **Name:** `devops-app`
   - **AMI:** Ubuntu 22.04 LTS
   - **Instance Type:** `t2.micro` (free tier eligible)
   - **Key Pair:** Create and download new key pair
   - **Storage:** 20 GB (default is fine)
3. Click **"Launch Instance"**
4. Wait 2-3 minutes for instance to start
5. Note the **Public IP address**

#### Step 2: Configure Security Group

1. Go to EC2 → Security Groups
2. Find the security group for your instance
3. Edit **Inbound Rules**
4. Add rule:
   - **Type:** HTTP
   - **Port:** 80
   - **Source:** 0.0.0.0/0 (open to internet)
5. Save

#### Step 3: Connect via SSH

**On your local machine:**

```bash
chmod 600 new-key.pem
ssh -i new-key.pem ubuntu@50.17.47.45
```

**You should now be inside the EC2 instance.**

#### Step 4: Install Docker and Docker Compose

**Inside EC2 instance:**

```bash
sudo apt update
sudo apt install docker.io docker-compose git -y
sudo usermod -aG docker ubuntu
```

**Exit and reconnect:**
```bash
exit
ssh -i new-key.pem ubuntu@50.17.47.45
```

**Verify installation:**
```bash
docker --version
docker-compose --version
git --version
```

#### Step 5: Deploy Application

**Inside EC2 instance:**

```bash
git clone https://github.com/YOUR_USERNAME/devops.git
cd devops
docker-compose up -d --build
```

**Wait for build to complete (2-3 minutes).**

**Verify containers are running:**
```bash
docker-compose ps
```

Should show:
- `chat-nginx` - running
- `chat-backend` - running

#### Step 6: Access Application

**Open browser:**
http://50.17.47.45
**Test multi-tab chat:**
- Open in two browser tabs
- Send message from Tab 1
- Verify it appears in Tab 2
- Test both directions

### CI/CD Setup (Automated Deployment)

#### Step 1: Fork Repository on GitHub

1. Go to `https://github.com/yaswanthsai257/devops`
2. Click **Fork** button
3. Creates copy under your account

#### Step 2: Update Local Git Remote

```bash
cd ~/devops
git remote remove origin
git remote add origin https://github.com/YOUR_USERNAME/devops.git
git push -u origin main
```

#### Step 3: Add GitHub Secrets

1. Go to your GitHub repo → **Settings** → **Secrets and variables** → **Actions**
2. Click **"New repository secret"**

**Add three secrets:**

**Secret 1: SSH_PRIVATE_KEY**
- Name: `SSH_PRIVATE_KEY`
- Value: Contents of your new-key.pem file (paste entire file)

**Secret 2: SSH_HOST**
- Name: `SSH_HOST`
- Value: `50.17.47.45`

**Secret 3: SSH_USER**
- Name: `SSH_USER`
- Value: `ubuntu`

#### Step 4: Deployment File Already Included

The `.github/workflows/deploy.yml` file is already in the repository.

**Verify it exists:**
```bash
cat .github/workflows/deploy.yml
```

#### Step 5: Test Automation

**Make a small change:**
```bash
cd ~/devops
echo "# Testing CI/CD" >> README.md
git add README.md
git commit -m "Test CI/CD automation"
git push origin main
```

**Watch deployment:**
1. Go to GitHub repo → **Actions** tab
2. See workflow running
3. Wait for green checkmark (success) or red X (failure)
4. After 1-2 minutes, check http://50.17.47.45 to verify new version is deployed

**Every future push will automatically redeploy!**

---

## Testing Multi-User Chat

### Step-by-Step Test

**Verify WebSocket Works:**

1. **Open browser tab 1:**
http://50.17.47.45
2. **Open browser tab 2:**
http://50.17.47.45
3. **In Tab 1, send a message** (type and press Enter)

4. **Check Tab 2** - message should appear immediately

5. **In Tab 2, send a message**

6. **Check Tab 1** - message should appear immediately

7. **Open browser tab 3, repeat** - confirm all tabs sync

### What This Proves

- Frontend loads correctly ✓
- WebSocket connection established ✓
- Backend receives messages ✓
- Backend broadcasts to all clients ✓
- Nginx reverse proxy working ✓
- Docker networking working ✓

### Browser Console Check

**Press F12 to open Developer Tools:**

1. Click **Console** tab
2. Should see no errors (red messages)
3. May see connection logs (normal)
4. WebSocket errors would indicate configuration issues

---

## Troubleshooting

### Issue: App shows 404 Not Found

**Symptoms:**
- Browser shows 404 error when accessing 50.17.47.45
- Frontend never loads

**Causes:**
- Frontend volume not mounted
- Nginx directory wrong
- Containers not running

**Fix:**
```bash
# SSH into EC2
ssh -i new-key.pem ubuntu@50.17.47.45

# Check if containers running
docker-compose ps

# Check Nginx container logs
docker-compose logs nginx

# Verify frontend directory mounted
docker exec chat-nginx ls /usr/share/nginx/html

# If empty, check docker-compose.yml
cat docker-compose.yml | grep -A 5 volumes:
```

**Solution:**
- Ensure `./frontend:/usr/share/nginx/html:ro` in docker-compose.yml
- Rebuild: `docker-compose down && docker-compose up -d --build`

---

### Issue: Chat Doesn't Update in Real-Time

**Symptoms:**
- Messages don't appear in other tabs
- No errors in browser console
- App loads fine

**Causes:**
- WebSocket upgrade headers commented out
- Backend not receiving WebSocket connection

**Fix:**
```bash
# SSH into EC2
ssh -i new-key.pem ubuntu@50.17.47.45

# Check nginx.conf
cat nginx.conf | grep -A 3 "proxy_set_header Upgrade"

# Should see (uncommented):
# proxy_set_header Upgrade $http_upgrade;
# proxy_set_header Connection "upgrade";

# If commented, edit and rebuild
docker-compose down
docker-compose up -d --build
```

---

### Issue: Cannot Connect to Backend

**Symptoms:**
- Browser shows 502 Bad Gateway
- Nginx logs show "connection refused"

**Causes:**
- Backend not binding to 0.0.0.0
- Backend not running
- Nginx using wrong hostname

**Fix:**
```bash
# SSH into EC2
ssh -i new-key.pem ubuntu@50.17.47.45

# Check backend logs
docker-compose logs backend

# Should see "Uvicorn running on 0.0.0.0:8000"

# Check if backend listening on all interfaces
docker exec chat-backend netstat -tuln | grep 8000

# Should show 0.0.0.0:8000

# Verify Dockerfile has --host 0.0.0.0
cat Dockerfile | grep "CMD"
```

---

### Issue: CI/CD Fails to Connect

**Symptoms:**
- GitHub Actions shows red X
- Error: "ssh: no key found" or "permission denied"

**Causes:**
- SSH_PRIVATE_KEY secret missing or wrong format
- SSH_HOST secret missing or wrong IP (should be 50.17.47.45)
- EC2 security group blocks SSH (port 22)

**Fix:**
```bash
# Verify SSH secrets in GitHub
# Settings → Secrets → Check all three are present

# Verify EC2 security group allows SSH (port 22)
# AWS EC2 → Security Groups → Inbound Rules
# Should have SSH (port 22) for 0.0.0.0/0

# Test SSH manually from local machine
ssh -i new-key.pem ubuntu@50.17.47.45
# Should work

# If SSH works manually but GitHub Actions fails
# Re-add SSH_PRIVATE_KEY secret (copy full key content carefully)
```

---

### Issue: Containers Keep Crashing

**Symptoms:**
- Containers show "Exited" status
- App was working, now broken

**Causes:**
- Out of disk space
- Out of memory
- Port already in use
- Config errors

**Fix:**
```bash
# SSH into EC2
ssh -i new-key.pem ubuntu@50.17.47.45

# Check container status
docker-compose ps

# View detailed logs
docker-compose logs

# Check disk space
df -h

# Check memory
free -h

# Rebuild from scratch
docker-compose down
docker system prune -a
docker-compose up -d --build
```

---

## Technology Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Frontend | HTML/CSS/JavaScript | Standard |
| Backend | Python FastAPI | 3.11 |
| Reverse Proxy | Nginx | Alpine |
| Containerization | Docker | Latest |
| Orchestration | Docker Compose | 3.8+ |
| Automation | GitHub Actions | Latest |
| Cloud | AWS EC2 | Ubuntu 22.04 |

## Project Files
devops/
├── .github/
│ └── workflows/
│ └── deploy.yml # CI/CD automation
├── app/
│ ├── main.py # FastAPI backend
│ └── requirements.txt # Python dependencies
├── frontend/
│ └── index.html # Chat UI
├── Dockerfile # Backend container
├── docker-compose.yml # Container orchestration
├── nginx.conf # Reverse proxy config
└── README.md # This file
## Key Learnings

1. **Container networking** - Containers communicate via service names on Docker networks
2. **Reverse proxy** - Nginx acts as single entry point to multiple backend services
3. **WebSocket proxying** - Requires special headers (Upgrade, Connection)
4. **Port binding** - Containers must bind to 0.0.0.0, not localhost
5. **CI/CD automation** - GitHub Actions can deploy on every push
6. **SSH security** - Private keys must be protected (GitHub Secrets)
7. **Docker networking** - Multiple containers can communicate on same network

## Deployment Checklist

- [x] EC2 instance launched and running (50.17.47.45)
- [x] Security group allows HTTP (port 80)
- [x] Docker and Docker Compose installed
- [x] Repository cloned on EC2
- [x] `docker-compose up -d --build` succeeds
- [x] App loads at `http://50.17.47.45`
- [x] Multi-tab chat works
- [x] GitHub repository forked
- [x] SSH secrets added to GitHub
- [x] CI/CD workflow file present
- [x] Test push triggers automated deployment

---

## Additional Resources

- [Docker Documentation](https://docs.docker.com/)
- [Nginx Reverse Proxy Guide](https://nginx.org/en/docs/)
- [FastAPI WebSocket Docs](https://fastapi.tiangolo.com/advanced/websockets/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)

