# FastAPI → Docker → Jenkins → AWS EC2 Deployment Guide

Complete end-to-end guide to deploy the Products FastAPI app to AWS EC2 using Docker and a Jenkins CI/CD pipeline.

**Final architecture:**
```
GitHub push
   │
   ▼
Jenkins (build → test → push image → SSH deploy)
   │
   ▼
Docker Hub (image registry)
   │
   ▼
AWS EC2 (Ubuntu + Docker) ── exposes port 80 ──► Public Internet
```

---

## Prerequisites

- AWS account (you have this)
- Docker Hub account → https://hub.docker.com (free)
- GitHub account (to host the code and trigger Jenkins)
- Git installed locally
- The project files: `main.py`, `models.py`, `requirements.txt`, `Dockerfile`, `.dockerignore`, `Jenkinsfile`

---

## Part 1 — Build & Test Docker Locally

### 1.1 Install Docker Desktop (Windows)
Download from https://www.docker.com/products/docker-desktop/ and install. Launch Docker Desktop and wait until the whale icon says "Docker Desktop is running".

### 1.2 Build the image
Open PowerShell in the project folder:
```powershell
cd "C:\Users\AtharvSingh\Desktop\ME\PYTHON\FAST_API\deploy\01-products-get-post"
docker build -t fastapi-products:local .
```

### 1.3 Run the container
```powershell
docker run -d --name fastapi-test -p 8000:8000 fastapi-products:local
```

### 1.4 Verify
Open http://127.0.0.1:8000/docs — you should see Swagger UI.

### 1.5 Stop & clean up
```powershell
docker rm -f fastapi-test
```

---

## Part 2 — Push Code to GitHub

```powershell
cd "C:\Users\AtharvSingh\Desktop\ME\PYTHON\FAST_API\deploy\01-products-get-post"
git init
git add .
git commit -m "Initial FastAPI deployment setup"
```

Create a new repo on GitHub (e.g. `fastapi-products`), then:
```powershell
git branch -M main
git remote add origin https://github.com/<your-username>/fastapi-products.git
git push -u origin main
```

---

## Part 3 — Launch AWS EC2 Instance

### 3.1 Sign in to AWS Console → EC2 → Launch Instance

| Setting | Value |
|---|---|
| **Name** | `fastapi-server` |
| **AMI** | Ubuntu Server 22.04 LTS (Free tier) |
| **Instance type** | `t3.micro` (free tier) or `t3.small` |
| **Key pair** | Create new → name `fastapi-key` → download `.pem` |
| **Network — Allow** | ✅ SSH (22), ✅ HTTP (80), ✅ HTTPS (443) |
| **Storage** | 8 GB gp3 (default) |

Click **Launch Instance**. Wait until **Instance state = Running** and copy the **Public IPv4 address** (e.g. `3.110.xxx.xxx`).

### 3.2 Set `.pem` permissions (Windows PowerShell)
```powershell
icacls "C:\path\to\fastapi-key.pem" /inheritance:r
icacls "C:\path\to\fastapi-key.pem" /grant:r "$($env:USERNAME):(R)"
```

### 3.3 SSH into the instance
```powershell
ssh -i "C:\path\to\fastapi-key.pem" ubuntu@<EC2_PUBLIC_IP>
```

### 3.4 Install Docker on EC2
Run on the EC2 instance:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu
exit
```
Reconnect via SSH (so the new group takes effect) and verify:
```bash
docker --version
docker ps
```

---

## Part 4 — Install & Configure Jenkins

You have two options — pick one.

### Option A — Jenkins on a second EC2 instance (recommended for production)
Launch another `t3.small` Ubuntu instance (same way as Part 3), open port **8080** in its security group, SSH in, then run:

```bash
# Install Java + Jenkins
sudo apt update
sudo apt install -y fontconfig openjdk-17-jre
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins

# Install Docker on the Jenkins box (Jenkins needs it to build images)
sudo apt install -y docker.io
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Access Jenkins at `http://<JENKINS_EC2_IP>:8080`. Unlock with:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Install **Suggested Plugins** and create your admin user.

### Option B — Jenkins locally (for learning)
Install via: https://www.jenkins.io/download/ → run on `http://localhost:8080`. Only works if your local machine is reachable from your EC2 host (or you only use it to trigger builds manually).

### 4.1 Install required Jenkins plugins
**Manage Jenkins → Plugins → Available plugins** — install:
- Docker Pipeline
- SSH Agent
- Git
- Pipeline

### 4.2 Add credentials
**Manage Jenkins → Credentials → System → Global credentials → Add**:

1. **Docker Hub login**
   - Kind: *Username with password*
   - Username: your Docker Hub username
   - Password: your Docker Hub password (or access token — recommended)
   - ID: `dockerhub-creds` ← must match the Jenkinsfile

2. **EC2 SSH key**
   - Kind: *SSH Username with private key*
   - Username: `ubuntu`
   - Private key: paste contents of `fastapi-key.pem`
   - ID: `ec2-ssh-key` ← must match the Jenkinsfile

---

## Part 5 — Wire Up the Jenkins Pipeline

### 5.1 Edit the Jenkinsfile
Open `Jenkinsfile` in your repo and update these three lines:
```groovy
DOCKERHUB_USER = 'your-dockerhub-username'
IMAGE_NAME     = 'fastapi-products'
EC2_HOST       = 'ubuntu@3.110.xxx.xxx'   // your APP EC2 public IP
```
Commit & push:
```powershell
git add Jenkinsfile
git commit -m "Configure Jenkinsfile for my AWS/Docker Hub"
git push
```

### 5.2 Create the pipeline job in Jenkins
**New Item → `fastapi-products-pipeline` → Pipeline → OK**

Configure:
- **Pipeline → Definition:** *Pipeline script from SCM*
- **SCM:** Git
- **Repository URL:** `https://github.com/<you>/fastapi-products.git`
- **Branch:** `*/main`
- **Script Path:** `Jenkinsfile`

Click **Save**.

### 5.3 Run it
Click **Build Now**. The pipeline will:
1. Clone the repo
2. Build the Docker image
3. Smoke-test it
4. Push it to Docker Hub
5. SSH into EC2 and `docker run` the new image on port 80
6. Verify with `curl`

### 5.4 Verify from your browser
Open `http://<EC2_PUBLIC_IP>/` — you should see `"Hello world"`.
Also try `http://<EC2_PUBLIC_IP>/docs` for Swagger UI.

---

## Part 6 — Automate Builds on Git Push (Optional)

Enable GitHub → Jenkins webhook:

1. In Jenkins job → **Configure → Build Triggers** → check **GitHub hook trigger for GITScm polling**.
2. On GitHub repo → **Settings → Webhooks → Add webhook**:
   - Payload URL: `http://<JENKINS_EC2_IP>:8080/github-webhook/`
   - Content type: `application/json`
   - Trigger: *Just the push event*

Now every `git push` triggers a build and redeploys automatically.

---

## Part 7 — Useful Debugging Commands

### On the EC2 app server
```bash
docker ps                          # see running containers
docker logs fastapi-products       # app logs
docker logs -f fastapi-products    # follow logs
docker restart fastapi-products    # restart
docker exec -it fastapi-products sh  # shell into container
```

### On the Jenkins server
```bash
sudo systemctl status jenkins
sudo journalctl -u jenkins -f      # live Jenkins logs
sudo tail -f /var/log/jenkins/jenkins.log
```

### Common issues
| Problem | Fix |
|---|---|
| `permission denied` on docker socket | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| EC2 port 80 not reachable | Check Security Group allows HTTP from `0.0.0.0/0` |
| Container exits immediately | `docker logs fastapi-products` → fix error, rebuild |
| Jenkins can't SSH to EC2 | Verify `ec2-ssh-key` credential has the correct private key and username `ubuntu` |

---

## Part 8 — Cost & Cleanup

**Approximate monthly cost** (if you leave everything running):
- App EC2 `t3.micro`: ~free (first year) or ~$7.50
- Jenkins EC2 `t3.small`: ~$15
- Data transfer / storage: negligible for this app

**To avoid charges**, stop or terminate instances when not using them:
- **Stop** = pause (no compute charge, small storage charge)
- **Terminate** = delete permanently

```bash
# Via AWS CLI
aws ec2 stop-instances --instance-ids i-xxxxxxxxxxxxx
```

---

## Summary Checklist

- [ ] Docker Desktop installed; built & ran locally
- [ ] Code pushed to GitHub
- [ ] App EC2 launched; Docker installed on it
- [ ] Security group allows 22/80/443
- [ ] Jenkins EC2 (or local) running with Docker + required plugins
- [ ] `dockerhub-creds` and `ec2-ssh-key` credentials added to Jenkins
- [ ] Jenkinsfile updated with your usernames & IP
- [ ] Pipeline job created from GitHub
- [ ] First build succeeds → app reachable at `http://<EC2_IP>/`
- [ ] (Optional) GitHub webhook set up for auto-deploy
