---
name: continuous-deployment
description: "Use when setting up automated deployment from GitHub Actions to a VM, when the user asks for CD or continuous deployment, when a project has CI but no deploy pipeline, or when Kamal setup is needed."
---

# Continuous Deployment with Kamal

Sets up continuous deployment from GitHub Actions to the VM that Claude Code is running on, using Kamal for orchestration and GHCR for container images.

## Prerequisites

These must be completed before starting:

- **configuring-github-credentials** — gh CLI authenticated, git configured
- **project-quality-setup** — CI workflow exists in `.github/workflows/ci.yml`
- A project repo cloned on the VM

## Progress checklist

```
- [ ] Clear port conflicts (Apache, nginx, etc.)
- [ ] Install Docker on the VM
- [ ] Install Ruby and Kamal on the VM
- [ ] Generate Dockerfile (if missing)
- [ ] Generate Kamal config (config/deploy.yml)
- [ ] Generate deploy SSH key and set GitHub secrets
- [ ] Run initial deploy via kamal setup on the VM
- [ ] Create GitHub Actions deploy workflow
- [ ] Verify full pipeline end-to-end
```

## Step 1: Clear port conflicts

Kamal's proxy needs ports 80 and 443. Check for existing services and stop them:

```bash
ss -tlnp | grep -E ':80\b|:443\b'
```

If Apache is found:

```bash
systemctl stop apache2 && systemctl disable apache2
```

If nginx is found:

```bash
systemctl stop nginx && systemctl disable nginx
```

Check for any other services on these ports and stop/disable them. Verify ports are free before proceeding:

```bash
ss -tlnp | grep -E ':80\b|:443\b' && echo "PORTS STILL IN USE" || echo "PORTS FREE"
```

**Do this once on the VM now.** Do not defer to the GitHub Actions workflow.

## Step 2: Install Docker on the VM

Check if Docker is installed, install if missing:

```bash
command -v docker && echo "INSTALLED" || echo "MISSING"
```

If missing, detect the distro and install:

```bash
. /etc/os-release && echo "Distro: $ID"
curl -fsSL https://get.docker.com | sh
```

Verify:

```bash
docker info
```

## Step 3: Install Ruby and Kamal on the VM

Check if Ruby is installed, install if missing using the distro package manager. Then install Kamal:

```bash
command -v ruby && echo "INSTALLED" || echo "MISSING"
```

If Ruby is missing, install it (e.g., `apt-get install -y ruby` on Debian/Ubuntu).

Then install Kamal:

```bash
gem install kamal
kamal version
```

Kamal must be installed on the VM so you can run `kamal setup` locally for the initial deploy and debug issues.

## Step 4: Generate Dockerfile

If the project already has a `Dockerfile`, skip this step.

Otherwise, detect the tech stack from the project root (`package.json`, `go.mod`, `Gemfile`, `Cargo.toml`, etc.) and generate a production Dockerfile:

- Use multi-stage builds for smaller final images
- Install only production dependencies
- Set appropriate `EXPOSE` port
- Use a minimal base image (alpine variants where possible)

Also create a `.dockerignore` if missing (exclude `.git`, `node_modules`, etc.).

Verify the image builds:

```bash
docker build -t <app>:test .
```

## Step 5: Generate Kamal config

### Detect domain

Check for Clodhost CNAME first:

```bash
hostname -f
```

If the hostname matches `*.clodhost.com`, default to that as the domain. Otherwise, ask the user for their domain name.

Ask the user if they want to use a custom domain instead of the detected default.

### Detect public IP

```bash
curl -s ifconfig.me
```

### Ask the user

- App service name (default: repo name)
- Domain confirmation

### Generate `config/deploy.yml`

```yaml
service: <app-name>
image: ghcr.io/<owner>/<repo>

servers:
  web:
    - <VM_PUBLIC_IP>

proxy:
  ssl: true
  host: <domain>
  app_port: <port>

registry:
  server: ghcr.io
  username:
    - KAMAL_REGISTRY_USERNAME
  password:
    - KAMAL_REGISTRY_PASSWORD

ssh:
  user: root

builder:
  arch: amd64
```

Set `app_port` to match the `EXPOSE` port in the Dockerfile.

### Environment variables

Check if the app needs env vars (look for `.env.example`, `.env.sample`, framework config files). If yes:

1. List the required env var names
2. Tell the user to set the values as GitHub Actions secrets via the GitHub UI: `Settings > Secrets and variables > Actions`
3. Configure the env vars in `config/deploy.yml`:

```yaml
env:
  clear:
    NODE_ENV: production
  secret:
    - DATABASE_URL
    - SECRET_KEY
```

**Do NOT ask the user to paste secret values into the terminal.** Direct them to the GitHub UI.

Add `.kamal/secrets` to `.gitignore`.

## Step 6: Deploy SSH key and GitHub secrets

### Generate a dedicated deploy key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/deploy_ed25519 -N "" -C "kamal-deploy"
```

Add the public key to authorized_keys:

```bash
cat ~/.ssh/deploy_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Set GitHub Actions secrets

Detect the repo from the git remote:

```bash
gh repo view --json nameWithOwner --jq .nameWithOwner
```

Show the user a summary of what secrets will be set:

| Secret | Description |
|--------|-------------|
| `VPS_SSH_KEY` | Deploy private key for GitHub Actions to SSH into the VM |
| `VPS_HOST` | VM's public IP address |

Ask for confirmation, then set them:

```bash
gh secret set VPS_SSH_KEY < ~/.ssh/deploy_ed25519 -R <owner/repo>
echo "<VM_IP>" | gh secret set VPS_HOST -R <owner/repo>
```

**Note:** `KAMAL_REGISTRY_PASSWORD` is NOT set as a secret — the workflow uses `${{ secrets.GITHUB_TOKEN }}` directly.

## Step 7: Initial deploy via `kamal setup`

This validates the entire setup before depending on GitHub Actions.

### Authenticate Docker to GHCR

```bash
echo $(gh auth token) | docker login ghcr.io -u $(gh api user --jq .login) --password-stdin
```

### Run kamal setup

```bash
KAMAL_REGISTRY_PASSWORD=$(gh auth token) KAMAL_REGISTRY_USERNAME=$(gh api user --jq .login) kamal setup
```

### Verify

```bash
curl -sI https://<domain>
```

Should return HTTP 200 with a valid certificate.

### If it fails

Debug with:

```bash
kamal app logs
kamal proxy logs
docker ps
```

Common issues:
- **Port still in use:** Revisit Step 1
- **DNS not pointing to VM:** Let's Encrypt will fail. Check `dig +short <domain>` matches the VM's IP. If DNS hasn't propagated, deploy without SSL first and add it after.
- **Dockerfile build errors:** Fix the Dockerfile and rebuild

## Step 8: Create GitHub Actions deploy workflow

Read the existing CI workflow to get its exact `name:` field:

```bash
head -5 .github/workflows/ci.yml
```

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  workflow_run:
    workflows: [<EXACT CI WORKFLOW NAME>]
    types: [completed]
    branches: [main]

concurrency:
  group: deploy
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: >
      github.event.workflow_run.conclusion == 'success' &&
      github.event.workflow_run.event == 'push'
    permissions:
      packages: write
      contents: read
    steps:
      - uses: actions/checkout@v6
        with:
          ref: ${{ github.event.workflow_run.head_sha }}

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.3"

      - name: Install Kamal
        run: gem install kamal

      - uses: docker/setup-buildx-action@v3

      - name: Set up SSH key
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.VPS_SSH_KEY }}

      - name: Configure SSH known hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan ${{ secrets.VPS_HOST }} >> ~/.ssh/known_hosts 2>/dev/null
          echo -e "Host *\n  StrictHostKeyChecking accept-new" >> ~/.ssh/config

      - name: Expose GitHub Actions cache env
        uses: crazy-max/ghaction-github-runtime@v3

      - name: Deploy with Kamal
        run: kamal deploy
        env:
          KAMAL_REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
          KAMAL_REGISTRY_USERNAME: ${{ github.actor }}
          KAMAL_SSH_HOST: ${{ secrets.VPS_HOST }}
```

**Critical details:**
- The `ref` in checkout is required — `workflow_run` triggers check out the default branch by default, not the commit that triggered CI
- The `workflows:` value must exactly match the `name:` field in ci.yml
- The `if:` condition ensures deploys only run for successful CI on push events (not PRs)
- `concurrency` prevents overlapping deploys
- `packages: write` is required for GHCR push
- `crazy-max/ghaction-github-runtime` enables Docker layer caching
- `docker/setup-buildx-action` enables efficient multi-platform builds

### Commit and push

Commit the Dockerfile, `config/deploy.yml`, `.github/workflows/deploy.yml`, and `.gitignore` changes together. Push to main.

## Step 9: Verify full pipeline

Monitor the CI and deploy workflows:

```bash
gh run watch
```

After the deploy workflow completes, verify:

```bash
curl -sI https://<domain>
```

Confirm:
- App responds on HTTPS with valid certificate
- `kamal app logs` shows healthy output
- `kamal proxy logs` shows no errors
- `docker ps` shows the container running

## Completion

Summarize what was set up:
- Dockerfile and Kamal config
- Deploy workflow gated on CI success
- HTTPS with Let's Encrypt auto-renewal
- List any app-specific env vars the user still needs to set in GitHub Actions secrets

Explain the deploy flow: **push to main → CI passes → auto-deploy**

Note useful Kamal commands:
- `kamal app logs` — view application logs
- `kamal proxy logs` — view proxy/SSL logs
- `kamal rollback` — roll back to previous version
- `kamal app exec` — run commands in the app container

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Apache/nginx still running on port 80 | Stop and disable in Step 1, not in the GitHub Actions workflow |
| `workflow_run.workflows` name doesn't match CI | Read the exact `name:` field from ci.yml |
| No local `kamal setup` before GitHub Actions | Always validate locally first — don't skip to the workflow |
| Asking user to paste secrets into terminal | Direct them to GitHub UI for app env vars |
| Using a PAT for GHCR | Use `GITHUB_TOKEN` in workflow, `gh auth token` for local |
| Forgetting `packages: write` permission | GHCR push will fail without it |
| No concurrency group | Overlapping deploys cause failures |
| DNS not pointing to VM | Let's Encrypt fails — check before initial deploy |
| Missing `ref` in checkout for `workflow_run` | Without `ref: ${{ github.event.workflow_run.head_sha }}`, checkout gets the wrong commit |
