# DevOps Documentation

Full-stack Hello World app (Django backend + React/Vite frontend), containerized with Docker, built and deployed automatically by Jenkins to an AWS EC2 server.

| Item | Value |
|---|---|
| Repository | https://github.com/Yash-choure/Full-stack-deployment-DevOps |
| Deployed URL | `http://<EC2_PUBLIC_IP>:5173` |
| Docker Hub images | `bunnyop/bunnyidv:bakend`, `bunnyop/bunnyidv:frontend` |
| Screenshot | Submitted separately in Drive |

## 1. Overview

```
git push -> Jenkins -> verify -> build images -> push to Docker Hub -> SSH to EC2 -> docker compose up -d
```

| Service | Stack | Port |
|---|---|---|
| backend | Django 6.0 REST API | 8000 |
| frontend | React (Vite, TypeScript) | 5173 |

```
.
├── backend/              Django app + Dockerfile
├── frontend/             React app + Dockerfile
├── docker-compose.yml    Runs both services
├── Jenkinsfile           CI/CD pipeline
└── terraform/            EC2 + security group (IaC)
```

## 2. Docker

- Each service has its own multi-stage Dockerfile: build tools stay in the build stage, only runtime files reach the final image.
- Both containers run as a non-root user.
- `.dockerignore` keeps `node_modules`, virtualenvs and `.git` out of the build.
- No secrets are stored in the repository. Settings are passed as environment variables.

| Variable | Service | Purpose |
|---|---|---|
| `ALLOWED_HOSTS` | backend | Hosts Django accepts requests for. `*` in compose; restrict to the server IP/domain in production. |

## 3. Run locally

**Prerequisites:** Docker 24+ with the Compose plugin, Git.

```bash
git clone https://github.com/Yash-choure/Full-stack-deployment-DevOps.git
cd Full-stack-deployment-DevOps
docker compose up -d
docker compose ps
```

**Verify**

| Check | Expected |
|---|---|
| `curl http://localhost:8000/api/hello/` | JSON "Hello" response |
| Open http://localhost:5173 | Page shows the message from the API |

**Useful commands**

```bash
docker compose logs -f        # follow logs
docker compose down           # stop and remove containers
```

**Build from source instead of pulling images**

```bash
docker build -t bunnyop/bunnyidv:bakend ./backend
docker build -t bunnyop/bunnyidv:frontend ./frontend
docker compose up -d
```

## 4. Provision the server (Terraform)

**Prerequisites:** Terraform 1.5+, AWS CLI configured (`aws configure`), an existing EC2 key pair.

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

This creates one Ubuntu EC2 instance and a security group that allows **only** these inbound ports:

| Port | Protocol | Purpose |
|---|---|---|
| 22 | SSH | Administration and Jenkins deploy |
| 80 | HTTP | Web traffic |
| 443 | HTTPS | Secure web traffic |

Note the public IP from the Terraform output. Remove everything with `terraform destroy`.

## 5. Prepare the server (one time)

SSH in: `ssh -i <key.pem> ubuntu@<EC2_PUBLIC_IP>`

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu        # log out and back in afterwards
sudo mkdir -p /opt/hello-app && sudo chown ubuntu:ubuntu /opt/hello-app
docker compose version                # confirm Compose is installed
```

**Manual deploy (what Jenkins automates)**

```bash
cd /opt/hello-app
# place docker-compose.yml here
docker compose pull && docker compose up -d
```

Open `http://<EC2_PUBLIC_IP>:5173` to see the app.

## 6. Set up Jenkins (one time)

**Requirements:** a Jenkins server with Docker installed and the `jenkins` user in the `docker` group (`sudo usermod -aG docker jenkins`, then restart Jenkins).

**Plugins:** Git, GitHub, Pipeline, SSH Agent, Credentials Binding.

**Credentials** (Manage Jenkins -> Credentials -> Global):

| ID | Type | Value |
|---|---|---|
| `dockerhub-creds` | Username with password | Docker Hub username + access token |
| `deploy-ssh-key` | SSH username with private key | User `ubuntu` + the EC2 private key |
| `deploy-host` | Secret text | EC2 public IP or DNS |

**Job**
1. New Item -> **Pipeline** -> *Pipeline script from SCM* -> Git -> repository URL -> script path `Jenkinsfile`.
2. Enable **GitHub hook trigger for GITScm polling**.
3. In GitHub: Settings -> Webhooks -> add `http://<JENKINS_URL>/github-webhook/`, content type `application/json`, event *push*.

## 7. CI/CD pipeline

Triggered on every push to the main branch.

| Stage | Action |
|---|---|
| Checkout | Gets code and tags the build with the short commit SHA |
| Verify | In parallel: backend `manage.py check` + tests, frontend `npm ci && npm run build` |
| Build images | Builds backend and frontend images |
| Push images | Logs in with `dockerhub-creds` and pushes to Docker Hub |
| Deploy | Copies `docker-compose.yml` to `/opt/hello-app` on EC2 over SSH, then `docker compose pull && up -d` |

To test: commit a small change, push, and watch the build in Jenkins. A green build means the new version is live on the server.

## 8. Troubleshooting Log

**Problem:** After deploying to EC2, the frontend loaded but the API returned `400 Bad Request` (`DisallowedHost`).

**Cause:** Django only serves requests whose `Host` header is in `ALLOWED_HOSTS`, and the server's public IP was not on the list.

**Fix:** Made `ALLOWED_HOSTS` configurable through an environment variable and set it in `docker-compose.yml`. The host list now changes per environment without rebuilding the image.

**Quick checks for common failures**

| Symptom | Fix |
|---|---|
| Jenkins: `permission denied` on `docker.sock` | Add `jenkins` to the `docker` group and restart Jenkins |
| Deploy stage: SSH fails | Check `deploy-ssh-key`, the `deploy-host` value and that port 22 is open |
| Page not reachable | Check the security group and run `docker compose ps` on the server |
| Pipeline not triggered | Check the webhook delivery in GitHub and that the job tracks the right branch |

## 9. Assumptions

- Deployment target is a single AWS EC2 Ubuntu instance; Jenkins runs on a separate machine.
- "Main branch" means the repository's default branch (`master`).
- Images are public on Docker Hub, so the server pulls them without logging in.
- Jenkins is used for CI/CD as the assignment requires, so there is no `.github/workflows/` directory.
