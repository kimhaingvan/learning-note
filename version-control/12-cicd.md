# CI/CD & Deployment Workflows

> **Category**: Version Control | **Level**: Intermediate

---

## CI/CD — Definitions

| Term | Definition |
|------|------------|
| **CI** — Continuous Integration | Clone code → build → run automated tests. Integrates every commit into a shared pipeline. |
| **CD** — Continuous Delivery | Build succeeds → artifact is ready → **manual step** required to deploy to production. |
| **CD** — Continuous Deployment | Build succeeds → **automatically deploys** to production. No manual confirmation needed. |

### Key Difference: Delivery vs Deployment

```
Continuous Integration
    └── Continuous Delivery    → artifact ready, manual deploy trigger
    └── Continuous Deployment  → artifact ready, auto deploy (fully automated)
```

### Three Core Pipeline Stages

| Stage | Purpose |
|-------|---------|
| **Build** | Compile, package the application |
| **Test** | Unit tests, integration tests, security scan, code analysis |
| **Deploy** | Deploy packaged artifact to target environment |

---

## GitHub / GitLab Actions

- Reference: [docs.github.com/en/actions](https://docs.github.com/en/actions)

---

## GitLab CI/CD

### Connecting a GitLab Runner

1. **Install GitLab Runner** on your server.
2. **Register the runner** with GitLab:
   - Go to **GitLab → Settings → CI/CD → Runners** → copy the URL and token.
   - On the server: `gitlab-runner register` → enter URL and token → set description (use server name).
3. **Choose an executor:**
   - `docker` — runs jobs inside Docker containers
   - `shell` — runs jobs directly on the server shell
   - `kubernetes` — runs jobs as pods in a Kubernetes cluster
4. **Start the runner:**
   ```bash
   gitlab-runner run \
     --working-directory /home/gitlab-runner \
     --config /etc/gitlab-runner/config.toml \
     --user gitlab-runner
   ```

> **Note:** If you use `only: - tags`, the branch you're running CI on must be a **protected branch** and must have **protected tags** enabled.

### Writing `.gitlab-ci.yml`

```yaml
stages:
  - build
  - deploy
  - checklog

build:
  stage: build
  script:
    - echo "Building..."
    - npm install
    - npm run build
  artifacts:
    paths:
      - dist/

deploy:
  stage: deploy
  script:
    - echo "Deploying..."
    - cp -r dist/ /var/www/html/
  only:
    - main

checklog:
  stage: checklog
  script:
    - journalctl -u nginx --no-pager -n 50
```

---

## Deployment Environments

Projects should maintain at least **3 environments**:

| Environment | Purpose |
|-------------|---------|
| **Development** | Active development and feature testing |
| **Pre-production (Staging)** | QA, integration testing, client demos |
| **Production** | Live system for end users |

> You can have **multiple staging environments** (e.g., staging-1, staging-2).

---

## Deployment Strategies

### By Target Type

| Strategy | Description |
|----------|-------------|
| **Service** | Deploy as a system service (e.g., systemd) |
| **Container (standalone)** | Deploy as a single Docker container |
| **Container orchestration** | Deploy via Kubernetes (K8s) or similar tool |

### By Server Architecture

| Approach | Description |
|----------|-------------|
| **Same server** | Build and deploy happen on the same machine |
| **Separate servers** | Build on one server, deploy to staging/production on others |

---

## Workflow 1 — Simple Single-Server Deployment

1. GitLab Runner runs on the **same server** as the app.
2. Pipeline: clone → build → produce output files → deploy via nginx (or relevant server).
3. `.gitlab-ci.yml` runs everything in sequence.

**Remember:** Protected branches + protected tags must match your `only:` rules.

---

## Workflow 2 — Artifacts with Separate Servers

### Why Use Artifacts?

| Problem with Workflow 1 | Solution |
|-------------------------|----------|
| Can't rollback without re-running CI | Store versioned artifacts |
| Old source files are overwritten on each deploy | Each server pulls a specific build version |
| Can't monitor performance history across versions | Artifacts store each deploy as a versioned snapshot |

### How it Works

1. **Build server** produces an artifact (Docker image or binary package) and pushes it to an artifact registry (e.g., Docker Hub, JFrog Artifactory).
2. **Deploy servers** (staging, production) pull the exact artifact version they need.

### Setting Up a Domain for JFrog via nginx

```nginx
server {
    listen 80;
    server_name jfrog.devopsedu.vn;

    client_max_body_size 500M;

    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_pass http://192.168.213.104:8082/;
        proxy_redirect off;
    }
}
```

---

## Workflow 3 — Docker-Based Deployment

Build a Docker image → push to a container registry → deploy servers pull and run the image.

```yaml
# Example .gitlab-ci.yml for Docker workflow
stages:
  - build
  - deploy

build:
  stage: build
  script:
    - docker build -t registry.example.com/myapp:$CI_COMMIT_SHORT_SHA .
    - docker push registry.example.com/myapp:$CI_COMMIT_SHORT_SHA

deploy_staging:
  stage: deploy
  script:
    - ssh deploy@staging "docker pull registry.example.com/myapp:$CI_COMMIT_SHORT_SHA"
    - ssh deploy@staging "docker stop myapp || true"
    - ssh deploy@staging "docker run -d --name myapp -p 80:3000 registry.example.com/myapp:$CI_COMMIT_SHORT_SHA"
  only:
    - develop
```

---

## DevSecOps Pipeline — Key Steps

| Step | Description |
|------|-------------|
| **Secure access to CI service** | Protect your CI/CD credentials and runner tokens |
| **Code Analysis** | Check code quality, formatting, bugs, and security on every commit |
| **Composition Analysis (SCA)** | Scan open-source dependencies for known vulnerabilities and license issues |
| **Build** | Compile and package the application |
| **Test** | Automated tests (unit, integration, E2E) |
| **Deploy** | Push artifact to target environment |

---

## Setting Up a VM Server (Static IP with VMware/VMware Fusion)

### Configure Static IP

Edit the netplan config file:
```bash
vi /etc/netplan/<config-file>
```

Key fields:

| Field | Purpose |
|-------|---------|
| `dhcp4: false` | Disable DHCP so the IP doesn't change on reboot |
| `addresses` | Your static IP address (e.g., `192.168.213.100/24`) |
| `gateway4` | IP of your network gateway |
| `nameservers.addresses` | DNS servers (e.g., `[8.8.8.8, 8.8.4.4]` for Google DNS) |

### NAT vs Bridge Network

| Mode | Behavior |
|------|----------|
| **NAT** | VM shares the host's IP. Other machines on the network **cannot** reach the VM directly via static IP. |
| **Bridge** | VM gets its own IP on the network. Other machines **can** reach services on the VM (e.g., JFrog) via the VM's static IP. |

---

## What to Learn Next

- [08-git-branching.md](08-git-branching.md) — Branch strategies for CI/CD (GitFlow, trunk-based)
- Reference: [docs.github.com/en/actions](https://docs.github.com/en/actions)
