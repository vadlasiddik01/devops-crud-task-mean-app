# MEAN CRUD App - Containerized CI/CD Deployment

This repository is productionized for the interview task:
- Dockerized Angular frontend and Node/Express backend
- MongoDB deployed via Docker Compose
- Nginx reverse proxy serving app on port `80`
- GitHub Actions pipeline for build, push to Docker Hub, and VM auto-deploy

## 1) Architecture

- `frontend` (Angular 15) is served by Nginx container on port `80`
- Nginx proxies `/api/*` requests to `backend:8080`
- `backend` (Node.js + Express + Mongoose) uses `MONGO_URI`
- `mongo` (official `mongo:7`) persists data via Docker volume

## 2) Repository Structure

- `backend/Dockerfile`
- `frontend/Dockerfile`
- `frontend/nginx.conf`
- `docker-compose.yml`
- `.github/workflows/ci-cd.yml`
- `.env.example`

## 3) Local Verification

```bash
cp .env.example .env
# Edit DOCKERHUB_USERNAME only if you want custom image names locally

docker compose up --build -d
```

Open:
- UI: `http://localhost`
- API health: `http://localhost/api/tutorials`

Stop:

```bash
docker compose down
```

## 4) Create GitHub Repository

```bash
git init
git add .
git commit -m "feat: containerize MEAN app with CI/CD and nginx reverse proxy"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

## 5) Docker Hub Preparation

Create these repositories in Docker Hub:
- `<your-dockerhub-username>/mean-crud-backend`
- `<your-dockerhub-username>/mean-crud-frontend`

## 6) Provision Ubuntu VM on AWS EC2 (Recommended)

EC2 setup:
- Launch EC2 instance: Ubuntu 22.04 LTS
- Instance type: `t3.small` (or at least 2 vCPU / 2-4 GB RAM)
- Create/select key pair (`.pem`) for SSH
- Security Group inbound rules:
  - `22` (SSH) from your IP only
  - `80` (HTTP) from `0.0.0.0/0`

Collect these values after launch:
- EC2 Public IPv4 (for `VM_HOST` secret)
- SSH username: `ubuntu` (for `VM_USER` secret)

Install Docker + Compose plugin on VM:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```

## 7) GitHub Actions Secrets

In GitHub repo -> `Settings -> Secrets and variables -> Actions`, add:

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
- `VM_HOST` (public IP/DNS)
- `VM_USER` (usually `ubuntu`)
- `VM_SSH_KEY` (private SSH key content)

## 8) Secret Management

Secrets are managed using:
- GitHub Secrets for CI/CD credentials
- `.env` file on VM for runtime configuration

No secrets are committed to the repository.

`.env` values used by runtime:

```env
DOCKERHUB_USERNAME=your_dockerhub_username
IMAGE_TAG=latest
MONGO_URI=mongodb://mongo:27017/dd_db
NODE_ENV=production
PORT=8080
```

## 9) CI/CD Workflow Behavior

Workflow file: `.github/workflows/ci-cd.yml`

On every push to `main`:
1. Build backend and frontend images
2. Push image tags (`latest` and short SHA) to Docker Hub
3. Copy `docker-compose.yml` to VM
4. SSH into VM, write `.env`, pull latest images, restart stack with `docker compose up -d`

## 10) Health & Verification

Basic service verification:
- Frontend reachable at `/`
- Backend API reachable at `/api/tutorials`
- MongoDB persistence via Docker volume

Containers can be inspected using:
```bash
docker compose ps
docker compose logs backend frontend mongo
```

## 11) Reverse Proxy Details

Nginx config is in `frontend/nginx.conf`:
- `/` -> serves Angular SPA
- `/api/` -> proxied to backend service `backend:8080`

Result: full application is reachable through VM public IP on port `80`.

## 12) Non-Goals (intentionally not used)

The following were intentionally out of scope for this assignment:
- Kubernetes orchestration
- Cloud-managed secret stores (Vault / AWS Secrets Manager)
- HTTPS / domain configuration
- Chose GitHub Actions because it’s natively integrated with GitHub, requires no extra infrastructure

These are typically introduced when scaling beyond a single VM or during later production hardening phases.

## 13) Final Deployment Checklist

1. Create a new GitHub repository and push this code to `main`.
2. Create Docker Hub repositories:
   - `<your-dockerhub-username>/mean-crud-backend`
   - `<your-dockerhub-username>/mean-crud-frontend`
3. Launch AWS EC2 Ubuntu VM using Section 6.
4. SSH into EC2 and install Docker + Compose plugin using Section 6 commands.
5. In GitHub Actions secrets, add:
   - `DOCKERHUB_USERNAME`
   - `DOCKERHUB_TOKEN`
   - `VM_HOST` (EC2 public IP/DNS)
   - `VM_USER` (`ubuntu`)
   - `VM_SSH_KEY` (content of your private key)
6. Push one new commit to `main` and wait for GitHub Actions CI/CD run to finish.
7. Verify app on `http://<EC2_PUBLIC_IP>` and verify API at `http://<EC2_PUBLIC_IP>/api/tutorials`.
8. Capture screenshots listed in this README and commit them under `docs/screenshots/`.
9. Share your GitHub repository link as the final deliverable.
