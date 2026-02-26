# Continuous Deployment Skill Design

## Goal

A skill that instructs Claude on setting up continuous deployment from GitHub Actions to the VM it's running on, using Kamal for orchestration and GHCR for container images.

## Decisions

- **Kamal-native with local bootstrapping**: Kamal owns deploy config. Claude runs `kamal setup` on the VM first to validate, then GitHub Actions handles subsequent deploys.
- **GHCR only**: GitHub Container Registry, authenticated via `GITHUB_TOKEN` (no PATs).
- **Dedicated deploy key**: Separate from the GitHub SSH key. Claude generates it and writes it to GitHub Actions secrets via `gh secret set`.
- **Auto-set secrets with confirmation**: Claude shows the user what secrets will be set, asks for confirmation, then writes them via `gh CLI`.
- **Detect and generate Dockerfile**: Claude detects the tech stack and writes a Dockerfile if missing. No baked-in templates.
- **Stop Apache/nginx automatically**: If port 80/443 is occupied, stop and disable the service.
- **Let's Encrypt via Kamal proxy**: Built-in SSL with auto-cert.
- **Clodhost domain detection**: Default to `<server>.clodhost.com` if detected, with option for custom domain.
- **App env vars via GitHub UI**: Claude lists required env var names but directs the user to set values in GitHub Actions secrets UI. No pasting secrets into the terminal.

## Skill Flow

### 1. Prerequisites & Port Conflict Resolution

**Assumes already done:**
- `configuring-github-credentials` (gh CLI authenticated, SSH keys, git configured)
- `project-quality-setup` (CI workflow exists)
- Project repo cloned on the VM

**Port conflicts:**
1. Check ports 80/443: `ss -tlnp | grep -E ':80|:443'`
2. If Apache found: `systemctl stop apache2 && systemctl disable apache2`
3. Handle other services (nginx, etc.) the same way
4. Verify ports are free

### 2. Install Kamal & Docker on the VM

**Docker:**
1. Detect distro
2. Install via `curl -fsSL https://get.docker.com | sh`
3. Verify: `docker info`

**Kamal:**
1. Install Ruby if missing
2. `gem install kamal`
3. Verify: `kamal version`

### 3. Generate Dockerfile & Kamal Config

**Dockerfile (if missing):**
- Detect tech stack from project root
- Generate production Dockerfile (multi-stage, minimal final image)
- Add `.dockerignore`

**Kamal config (`config/deploy.yml`):**
- Ask user for app service name (default: repo name)
- Domain detection: check for Clodhost CNAME (`*.clodhost.com`), default to it if found, otherwise ask user. Offer custom domain option either way.
- Generate config: service, image (`ghcr.io/<owner>/<repo>`), servers (VM public IP via `curl -s ifconfig.me`), proxy/SSL (Let's Encrypt), registry (ghcr.io with `KAMAL_REGISTRY_PASSWORD` env var), builder (buildx)

**Environment variables:**
- Claude checks if app needs env vars (`.env.example`, framework conventions)
- If yes: list required env var names, tell user to set them in GitHub Actions secrets UI
- Configure `config/deploy.yml` to reference those secrets via Kamal's `env.secret`

### 4. Deploy Key & GitHub Secrets

**Generate deploy SSH key:**
1. `ssh-keygen -t ed25519 -f ~/.ssh/deploy_ed25519 -N "" -C "kamal-deploy"`
2. Add public key to VM's `~/.ssh/authorized_keys`

**Set GitHub Actions secrets via `gh secret set`:**

| Secret | Source |
|--------|--------|
| `VPS_SSH_KEY` | Contents of `~/.ssh/deploy_ed25519` (private key) |
| `VPS_HOST` | VM's public IP |
| `KAMAL_REGISTRY_PASSWORD` | Not needed — uses `${{ secrets.GITHUB_TOKEN }}` in workflow |

Claude shows summary, asks confirmation, then sets via `gh secret set -R <owner/repo>`.

### 5. Initial Deploy via `kamal setup`

**Validate first:**
1. `docker build -t <app>:test .`
2. `kamal config`
3. Authenticate Docker to GHCR: `echo $(gh auth token) | docker login ghcr.io -u <user> --password-stdin`

**Deploy:**
1. `KAMAL_REGISTRY_PASSWORD=$(gh auth token) kamal setup`
2. Verify: `curl -sI https://<domain>` returns 200

**If it fails:**
- Debug with `kamal app logs`, `kamal proxy logs`, `docker ps`
- DNS warning: Let's Encrypt fails if domain doesn't resolve to VM IP. Offer to deploy without SSL first.

### 6. GitHub Actions Deploy Workflow

**`.github/workflows/deploy.yml`:**
- Trigger: `workflow_run` on CI workflow, `main` branch, `completed` type
- Condition: `conclusion == 'success' && event == 'push'`
- Concurrency group: `deploy`, no cancel-in-progress
- Permissions: `packages: write`, `contents: read`

**Steps:**
1. Checkout
2. Set up Ruby + `gem install kamal`
3. Docker Buildx
4. SSH agent with `VPS_SSH_KEY`
5. SSH known hosts for `VPS_HOST`
6. Expose GitHub Actions cache env
7. `kamal deploy` with `KAMAL_REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}`

**Commit and push** all files together. Monitor with `gh run watch`.

### 7. Verification & Completion

**Verify pipeline:**
- `gh run watch` — confirm deploy workflow succeeds
- `curl -sI https://<domain>` — app live on HTTPS

**Smoke test:**
- Valid HTTPS cert
- `kamal app logs` healthy
- `kamal proxy logs` clean
- Container running via `docker ps`

**Hand off:**
- Summarize setup
- List any remaining env vars user needs to set
- Explain flow: push to main → CI → auto-deploy
- Note useful commands: `kamal app logs`, `kamal rollback`, `kamal app exec`
