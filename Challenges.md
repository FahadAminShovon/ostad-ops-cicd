## Challenges Faced & Troubleshooting (What happened + how it was solved)

### 1) `fnm` works in one step but not in later workflow steps (GitHub-hosted runner)
**Symptom**
- `fnm install 24` / `fnm use 24` worked in the first step.
- Later steps used a different Node version (ex: `node -v` showed `v20.x`), or `fnm` wasn’t found.

**Why it happened**
- Each GitHub Actions step runs in a new shell process.
- The environment changes (PATH updates, fnm shims) done in one step don’t automatically persist to later steps.
- On GitHub runners, a system Node may already exist, so later steps can silently fall back to it.

**Fix**
- Persisted fnm’s binary path using:
  - `echo "$HOME/.local/share/fnm" >> "$GITHUB_PATH"`
- Still needed to initialize fnm per step:
  - `eval "$(fnm env)"` and then `fnm use 24`
- Later migrated to `actions/setup-node@v4` to avoid shell-init complexity entirely.

---

### 2) `fnm use 24` error: “We can't find the necessary environment variables… evaluate `fnm env`”
**Symptom**
- Deploy/test step failed with:
  - `error: We can't find the necessary environment variables to replace the Node version...`

**Why it happened**
- `fnm` requires shell environment initialization (`eval "$(fnm env)"`) to set up variables and PATH shims.
- Having fnm installed or listed in `.bashrc` isn’t enough in CI because `.bashrc` may not load.

**Fix**
- Added `eval "$(fnm env)"` in the same step before `fnm use 24`.
- Eventually replaced fnm with `actions/setup-node@v4`.

---

### 3) Self-hosted runner: `fnm` installed on EC2 but not available in workflow jobs
**Symptom**
- On EC2 via SSH: `fnm --version` worked.
- In GitHub Actions deploy job: `fnm` wasn’t found or `fnm use` failed.

**Why it happened**
- The runner is usually executed as a system service (systemd) in a non-interactive shell.
- Non-interactive shells often don’t load `.bashrc`, so PATH changes defined there don’t apply.

**Fix**
- For reliability, stopped relying on `.bashrc` and migrated deploy job to:
  - `actions/setup-node@v4` for Node version management
- Kept server provisioning separate (pm2/nginx installed on the server once).

---

### 4) Installing pm2 with `sudo` failed: `sudo: npm: command not found`
**Symptom**
- `sudo npm install -g pm2` returned:
  - `sudo: npm: command not found`

**Why it happened**
- Node/npm installed via fnm is user-scoped (`ubuntu`), not system-wide for root.
- `sudo` runs with a restricted PATH; root couldn’t see fnm’s npm binary.

**Fix**
- Installed pm2 without sudo:
  - `npm install -g pm2`
- Used `sudo` only for system packages/services (like nginx).

---

### 5) Git checkout showed “detached HEAD” and looked suspicious
**Symptom**
- After checkout: Git printed “detached HEAD state”.

**Why it happened**
- CI checked out a specific commit (`GITHUB_SHA`) rather than a branch name.
- This is normal behavior when you checkout a commit hash directly.

**Fix**
- Kept it as-is. Deploying a specific SHA is actually preferable: deterministic deployment.

---

### 6) Nginx config error: “duplicate default server for 0.0.0.0:80”
**Symptom**
- `sudo nginx -t` failed with:
  - `duplicate default server for 0.0.0.0:80`

**Why it happened**
- Both the default nginx site and my new site were configured as `default_server` on port 80.

**Fix**
- Disabled the default site:
  - `sudo rm -f /etc/nginx/sites-enabled/default`
- Kept only my `node-app` site enabled.
- Verified with:
  - `sudo nginx -t`

---

### 7) Still seeing default Nginx page after enabling my site
**Symptom**
- Browser still showed the Nginx default page even though `node-app` was enabled.

**Troubleshooting steps**
- Verified enabled sites:
  - `ls -la /etc/nginx/sites-enabled`
- Verified config loaded:
  - `sudo nginx -T` to inspect active config
- Verified app is running:
  - `npm run start` (server printed “running on port 3000”)

**Root cause**
- Nginx config was correct, but external traffic wasn’t reaching the correct port due to AWS networking (see next issue).

---

### 8) Port 3000 reachable publicly but port 80 not reachable
**Symptom**
- `http://<EC2_IP>:3000` worked
- `http://<EC2_IP>` (port 80) didn’t

**Why it happened**
- Nginx was listening on port 80, but AWS Security Group inbound rule did not allow port 80.
- Confirmed nginx listening using:
  - `sudo ss -ltnp | grep ':80'`

**Fix**
- Added inbound Security Group rule:
  - HTTP (TCP 80) allowed
- After that, port 80 worked and nginx successfully reverse-proxied to 3000.

---

### 9) Artifact naming mistake (upload action not found)
**Symptom**
- Workflow error:
  - `Unable to resolve action actions/upload-artifacts, repository not found`

**Why it happened**
- Action name typo. Correct action is singular:
  - `actions/upload-artifact`

**Fix**
- Updated to:
  - `uses: actions/upload-artifact@v4`
  - `uses: actions/download-artifact@v4`

---

### 10) General workflow stability improvements
**What improved reliability**
- Used `set -euo pipefail` to fail fast and avoid silent pipeline failures.
- Used `if: always()` on artifact upload so test output is available even if tests fail.
- Used `needs: test` to enforce deploy gating and guarantee artifact availability.
- Migrated to marketplace actions (`checkout`, `setup-node`, `upload/download-artifact`) for standardization and fewer environment issues.