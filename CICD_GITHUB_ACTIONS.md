# Phase 3 — CI/CD with GitHub Actions

## Overview
Phase 3 implements a full CI/CD pipeline using GitHub Actions with a two-branch strategy:
- `dev` branch → runs tests + builds staging image → staging VM deploys automatically
- `main` branch → runs tests + builds production image → production VM deploys automatically

The VM is on a private local network (192.168.221.142) so GitHub Actions cannot SSH in directly.
Solution: pull-based deployment — VM pulls new Docker images from Docker Hub every 5 minutes via cron.

```
Push to dev
    ↓
GitHub Actions: run tests → build :dev-latest → push to Docker Hub
    ↓
Staging VM cron: git pull → docker compose pull → restart web container
    ↓
Review site on staging VM

Merge to main
    ↓
GitHub Actions: run tests → build :latest → push to Docker Hub
    ↓
Production VM cron: git pull → docker compose pull → restart web container
    ↓
Live on production
```

---

## STEP 1: Create the dev branch

What this does:
The dev branch is where all daily development happens. Tests run automatically on every push to dev.
No deployment happens on dev — only on main. This protects production from broken code.

On Windows VS Code terminal:
```bash
git checkout main
git pull origin main
git checkout -b dev
git push origin dev

# Verify both branches exist
git branch -a

Expected output:

* dev
  main
  remotes/origin/dev
  remotes/origin/main
  remotes/origin/vm_code

Go to GitHub in browser and verify dev branch appears in the branch dropdown before continuing.

## STEP 2: Generate a dedicated SSH key for GitHub Actions

What this does:
GitHub Actions needs to SSH into the VM to trigger deployments. A dedicated key pair is generated
specifically for GitHub Actions — completely separate from the MobaXterm personal key. This way
if you ever need to revoke GitHub Actions access you delete only that key without affecting
your personal SSH access.

On VM via SSH:
bash
# Generate new ED25519 key pair specifically for GitHub Actions
# Do NOT enter a passphrase — press Enter twice
# GitHub Actions cannot type a passphrase interactively
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions

# Add the PUBLIC key to authorized_keys
# This tells the VM: trust whoever has the matching private key
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys

# Set correct permissions — SSH refuses to use files with wrong permissions
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh/

# Test the key works by SSHing locally
ssh -i ~/.ssh/github_actions username@192.168.221.142 "echo SSH key works"
# Must print: SSH key works

# Print the PRIVATE key — copy this ENTIRE output for GitHub Secrets
cat ~/.ssh/github_actions

Important: Copy the ENTIRE private key output including:

-----BEGIN OPENSSH PRIVATE KEY-----
... many lines of base64 ...
-----END OPENSSH PRIVATE KEY-----

---

## STEP 3: Add GitHub Secrets

What this does:
GitHub Secrets are encrypted environment variables stored in your repository.
GitHub Actions can read them during workflow runs but they are never printed in logs.
GitHub automatically redacts any secret value that appears in output.

Go to: Your_repo → Settings → Secrets and variables → Actions → New repository secret

Add these 6 secrets exactly:

| Secret Name | Value |
|---|---|
| VM_HOST | 192.168.221.142 |
| VM_USER | username |
| VM_SSH_KEY | Full private key content from Step 2 |
| VM_PROJECT_PATH | /path/to/project/root |
| DOCKER_USERNAME | username |
| DOCKER_PASSWORD | Docker Hub access token (not your password) |

For DOCKER_PASSWORD — create an access token at:
Docker Hub → Account Settings → Security → Access Tokens → New Token
Name it: github-actions-deploy

Secret names are case sensitive. DOCKER_USERNAME is not the same as docker_username.

---

## STEP 4: Write smoke tests and verify locally first

What this does:
Tests must pass locally before GitHub Actions runs them. This catches import errors and
missing environment variables before they cause confusing CI failures. Tests run using
dummy environment variables in a temporary venv.

On Windows VS Code — open portfolio_pages/tests.py and replace entirely:
```python
from django.test import TestCase, Client
from django.contrib.auth.models import User

class SmokeTests(TestCase):

    def setUp(self):
        self.client = Client()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )

    def test_homepage_loads(self):
        response = self.client.get('/')
        self.assertIn(response.status_code, [200, 301, 302])

    def test_admin_page_loads(self):
        response = self.client.get('/admin/')
        self.assertIn(response.status_code, [200, 302])

    def test_database_works(self):
        User.objects.create_user(username='dbtest', password='dbtest123')
        count = User.objects.count()
        self.assertGreaterEqual(count, 1)

    def test_authenticated_admin(self):
        self.client.login(username='testuser', password='testpass123')
        response = self.client.get('/admin/')
        self.assertIn(response.status_code, [200, 302])
```

Run locally to verify before pushing:
```powershell
python -m venv test_venv
test_venv\Scripts\activate
pip install -r requirements.txt

# Set dummy environment variables
$env:SECRET_KEY="test-secret-key-local"
$env:DEBUG="True"
$env:ALLOWED_HOSTS="localhost,127.0.0.1"
$env:TEST_DB="sqlite"
$env:DB_PASSWORD="testpassword"
$env:EMAIL="test@test.com"
$env:EMAIL_PASSWORD="testpassword"
$env:EMAIL_BACKEND="django.core.mail.backends.console.EmailBackend"
$env:EMAIL_HOST="smtp.gmail.com"
$env:EMAIL_PORT="587"
$env:LINKEDIN_URL="https://linkedin.com"
$env:FACEBOOK_URL="https://facebook.com"
$env:INSTAGRAM_URL="https://instagram.com"
$env:GITHUB_URL="https://github.com"

python manage.py test --verbosity=2

deactivate
Remove-Item -Recurse -Force test_venv
```

Expected: Ran 4 tests — OK

Push:
```bash
git add portfolio_pages/tests.py
git commit -m "Add smoke tests for CI pipeline"
git push origin dev
```

====>> ERRORS FACED:

Error: Tests tried to connect to PostgreSQL instead of SQLite even after setting TEST_DB=sqlite

Cause: A .env file existed on the Windows machine at E:\portfolio_site\portfolio\.env
python-decouple reads .env automatically when it starts. Even though TEST_DB=sqlite
was set as a PowerShell environment variable, python-decouple was reading the .env file
and Django was still loading the PostgreSQL DATABASES config.

The root cause was that the DATABASES section in settings.py was always using
python-decouple's config() to read PostgreSQL credentials — it had no way to
switch to SQLite regardless of what environment variables were set.

Fix: Added an if/else block to settings.py that checks os.environ.get('TEST_DB')
BEFORE python-decouple reads anything. os.environ reads directly from the process
environment — it is not affected by .env files. If TEST_DB=sqlite is set,
Django uses SQLite and never calls config() for database credentials at all.

```python
import os

if os.environ.get('TEST_DB') == 'sqlite':
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.sqlite3',
            'NAME': BASE_DIR / 'db.sqlite3',
        }
    }
else:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql',
            'NAME': config('DB_NAME', default='portfolio_db'),
            'USER': config('DB_USER', default='portfolio_user'),
            'PASSWORD': config('DB_PASSWORD'),
            'HOST': config('DB_HOST', default='db'),
            'PORT': config('DB_PORT', default='5432'),
        }
    }
```

This same if/else works correctly in both environments:
- GitHub Actions sets TEST_DB=sqlite in the workflow env block → SQLite used
- Production VM never sets TEST_DB → PostgreSQL used
- The .env file on the server is irrelevant because os.environ.get() 
  checks the process environment directly, not the .env file

Error 2: test_admin_page_loads failed — AssertionError: 302 != 200

Cause: Original test used assertEqual(response.status_code, 200) but Django returns 302
(redirect to login) for unauthenticated admin requests in some configurations.

Fix: Changed to assertIn(response.status_code, [200, 302]) to accept both valid responses.

Error 3: test_database_works failed — AssertionError: 0 not greater than or equal to 1

Cause: Test relied on setUp creating a user and then counting — but the count returned 0.
The test was creating a user in setUp but not in the test method itself.

Fix: Added explicit user creation inside the test method:
```python
User.objects.create_user(username='dbtest', password='dbtest123')
count = User.objects.count()
self.assertGreaterEqual(count, 1)
```

## STEP 5: Create GitHub Actions workflow — test job only first

What this does:
The workflow file lives at .github/workflows/deploy.yml. GitHub Actions automatically detects
and runs any YAML file in this directory. Starting with only the test job lets you verify
tests pass in the cloud before adding any Docker or deploy complexity.

Important: The .github folder sits outside the portfolio/ subfolder at the repo root.
This means working-directory must be set to portfolio for all run commands.

On Windows VS Code — create .github/workflows/deploy.yml:
```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - dev

jobs:

  test:
    name: Run Django Tests
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: portfolio

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run Django tests
        env:
          SECRET_KEY: "test-secret-key-for-ci-not-real"
          DEBUG: "True"
          ALLOWED_HOSTS: "localhost,127.0.0.1"
          TEST_DB: "sqlite"
          DB_PASSWORD: "testpassword"
          EMAIL_BACKEND: "django.core.mail.backends.console.EmailBackend"
          EMAIL_HOST: "smtp.gmail.com"
          EMAIL_PORT: "587"
          EMAIL: "test@test.com"
          EMAIL_PASSWORD: "testpassword"
          LINKEDIN_URL: "https://linkedin.com"
          FACEBOOK_URL: "https://facebook.com"
          INSTAGRAM_URL: "https://instagram.com"
          GITHUB_URL: "https://github.com"
        run: |
          python manage.py test --verbosity=2
```

Push:
```bash
git add .github/workflows/deploy.yml
git commit -m "Add CI workflow — test job only"
git push origin dev
```

Go to GitHub → Actions tab within 30 seconds. A workflow run should appear.

====>> ERRORS FACED:

Error 1: No workflow appeared in GitHub Actions tab after pushing

Cause 1: Three YAML syntax errors in the workflow file:
- `-main` instead of `- main` (missing space after dash in branches list)
- `runs_on` instead of `runs-on` (underscore instead of hyphen)
- `steps:` was indented at job level instead of inside the test job

YAML does not throw errors for wrong indentation — it just silently does nothing.
These errors caused GitHub to reject the workflow file without any notification.

Fix: Corrected all three syntax errors. YAML uses spaces not tabs, 2-space indentation,
and hyphens not underscores for multi-word keys.

Cause 2: Workflow file existed on dev branch but not on main branch.
GitHub Actions requires the workflow file to exist on the default branch (main) before
it will run on any branch.

Fix: Merged dev into main so the workflow file existed on main, then pushed a new
commit to dev to trigger the workflow.

Error 2: Workflow file not found — working in wrong directory

The .github folder was created inside portfolio/ subfolder on Windows initially.
It must be at the repo root — the same level where .git folder is, not inside portfolio/.

Fix: Moved .github/ to repo root. Verified with:
```bash
ls .github/workflows/deploy.yml
# Must work from repo root
```

Error 3: Node.js 20 deprecation warning

Output showed warning about actions/checkout@v4 and actions/setup-python@v5 targeting Node.js 20.
This is just a warning not an error — GitHub is upgrading runners. Workflow still runs correctly.
Will resolve itself when action maintainers release updated versions.

---

## STEP 7: Add deploy job and validate on dev first

What this does:
The deploy job is added to the workflow but the condition `if: github.ref == 'refs/heads/main'`
prevents it from running on dev branch. Pushing to dev validates the YAML structure and job
dependencies without triggering a real Docker build. The deploy job shows as grey SKIPPED on dev.

On Windows VS Code — update .github/workflows/deploy.yml with complete file:
```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - dev

jobs:

  test:
    name: Run Django Tests
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: portfolio

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run Django tests
        env:
          SECRET_KEY: "test-secret-key-for-ci-not-real"
          DEBUG: "True"
          ALLOWED_HOSTS: "localhost,127.0.0.1"
          TEST_DB: "sqlite"
          DB_PASSWORD: "testpassword"
          EMAIL_BACKEND: "django.core.mail.backends.console.EmailBackend"
          EMAIL_HOST: "smtp.gmail.com"
          EMAIL_PORT: "587"
          EMAIL: "test@test.com"
          EMAIL_PASSWORD: "testpassword"
          LINKEDIN_URL: "https://linkedin.com"
          FACEBOOK_URL: "https://facebook.com"
          INSTAGRAM_URL: "https://instagram.com"
          GITHUB_URL: "https://github.com"
        run: |
          python manage.py test --verbosity=2

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/dev'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push staging image
        uses: docker/build-push-action@v5
        with:
          context: portfolio
          push: true
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/portfolio:dev-latest
            ${{ secrets.DOCKER_USERNAME }}/portfolio:dev-${{ github.sha }}

      - name: Notify staging ready
        run: |
          echo "Staging image pushed: ${{ secrets.DOCKER_USERNAME }}/portfolio:dev-latest"
          echo "Staging VM will pull this image within 5 minutes via cron"

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push production image
        uses: docker/build-push-action@v5
        with:
          context: portfolio
          push: true
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/portfolio:latest
            ${{ secrets.DOCKER_USERNAME }}/portfolio:${{ github.sha }}

      - name: Notify production ready
        run: |
          echo "Production image pushed: ${{ secrets.DOCKER_USERNAME }}/portfolio:latest"
          echo "Production VM will pull this image within 5 minutes via cron"
```

Key points:
- `needs: test` — deploy only runs if test job passes
- `if: github.ref == 'refs/heads/main'` — production deploy only on main branch
- `if: github.ref == 'refs/heads/dev'` — staging deploy only on dev branch  
- `context: portfolio` — Dockerfile is inside portfolio/ subfolder not at repo root
- Two image tags per deploy — :latest for pulling, :sha for rollback

Push to dev and verify:
- Run Django Tests → green
- Deploy to Staging → green (builds :dev-latest)
- Deploy to Production → grey SKIPPED (correct — this is dev branch)

====>> ERRORS FACED:

Error: context: . caused build failure

The workflow originally used context: . which tells Docker to look for Dockerfile
at the repo root. But the Dockerfile is inside portfolio/ subfolder because the repo
structure is:

```
My_Portfolio_website/     ← repo root
└── portfolio/            ← Django project (Dockerfile is here)
    ├── manage.py
    ├── Dockerfile
    └── requirements.txt
```

Fix: Changed context: . to context: portfolio so Docker finds the Dockerfile correctly.

---

## STEP 8: Merge to main — first real Docker build

What this does:
Merging dev into main triggers the full production pipeline — tests run, then the deploy
job builds the Docker image and pushes it to Docker Hub with two tags:
- :latest — production VM always pulls this tag
- :abc1234 — git commit SHA for pinned rollback

On Windows:
```bash
git checkout main
git merge dev
git push origin main
```

Watch GitHub Actions — both jobs must show green:
- Run Django Tests → green
- Deploy to Production → green (3-5 minutes for first build)

Verify on Docker Hub:
Go to hub.docker.com → Repositories → iamkaushal20/portfolio
Should show :latest tag with recent timestamp.

---

## STEP 9: Update docker-compose.yml to use Docker Hub image

What this does:
The production and staging VMs need to pull pre-built images from Docker Hub instead
of building locally. Using the PORTFOLIO_IMAGE environment variable from .env means
the same docker-compose.yml works on both VMs with different image tags.

On Windows VS Code — update docker-compose.yml web service:
```yaml
web:
  image: ${PORTFOLIO_IMAGE}
  # Production .env: PORTFOLIO_IMAGE=iamkaushal20/portfolio:latest
  # Staging .env:    PORTFOLIO_IMAGE=iamkaushal20/portfolio:dev-latest
  # Same file, different behaviour per environment
```

Push:
```bash
git add portfolio/docker-compose.yml
git commit -m "Use PORTFOLIO_IMAGE variable — same compose file for all environments"
git push origin dev
git checkout main && git merge dev && git push origin main
```

On production VM — add PORTFOLIO_IMAGE to .env:
```bash
nano /srv/portfoliosite/portfolio/.env
# Add this line:
PORTFOLIO_IMAGE=iamkaushal20/portfolio:latest
```

On staging VM — add PORTFOLIO_IMAGE to .env:
```bash
PORTFOLIO_IMAGE=iamkaushal20/portfolio:dev-latest
```

---

## STEP 10: Set up cron deploy script on both VMs

What this does:
GitHub Actions cannot SSH into a private VM (192.168.x.x is unreachable from the internet).
Instead the VM polls Docker Hub every 5 minutes and deploys automatically if a new image exists.
This is called pull-based deployment — a valid real-world pattern for air-gapped environments.

On both VMs — create deploy.sh:
```bash
nano /srv/portfoliosite/portfolio/deploy.sh
```

Paste:
```bash
#!/bin/bash
set -e
LOGFILE="/var/log/portfolio-deploy.log"
TS=$(date '+%Y-%m-%d %H:%M:%S')
echo "[$TS] Starting deploy..." >> $LOGFILE
cd /srv/portfoliosite/portfolio
git pull origin main >> $LOGFILE 2>&1
docker compose pull web >> $LOGFILE 2>&1
docker compose up -d --no-deps web >> $LOGFILE 2>&1
docker compose exec -T web python manage.py migrate --noinput >> $LOGFILE 2>&1
echo "[$TS] Deploy complete." >> $LOGFILE
```

For staging VM — change git pull origin main to git pull origin dev

```bash
chmod +x /srv/portfoliosite/portfolio/deploy.sh
sudo touch /var/log/portfolio-deploy.log
sudo chown kaushal20:kaushal20 /var/log/portfolio-deploy.log

# Test manually before adding to cron
./deploy.sh
cat /var/log/portfolio-deploy.log
```

Add to crontab:
```bash
crontab -e
# Add:
*/5 * * * * /srv/portfoliosite/portfolio/deploy.sh
```

====>> ERRORS FACED:

Error 1: deploy.sh: command not found

Cause: Linux does not look in the current directory when you type a command.
Only directories in PATH are searched. The current directory is not in PATH by default.

Fix: Use explicit path: ./deploy.sh or full path /srv/portfoliosite/portfolio/deploy.sh
In cron always use the full absolute path.

Error 2: git pull — detached HEAD state

Cause: The VM was in detached HEAD state — not on any branch.
git pull had no local branch to merge into.

Fix:
```bash
git checkout main
git pull origin main
git branch
# Must show: * main
```

Error 3: Disk completely full (100%) during cron deploy

Cause: Docker images from multiple builds accumulated on the VM.
Each GitHub Actions push created a new image that was pulled onto the VM.
Old images were never cleaned up. Total Docker image usage exceeded available disk space.

```bash
df -h
# Showed: 9.8G used, 0 available, 100%
```

Fix: Emergency cleanup freed 3.5GB:
```bash
docker system prune -a
# Type y to confirm
```

Also removed git lock file left by the crashed git operation:
```bash
rm /srv/portfoliosite/.git/index.lock
```

Added weekly cleanup cron to prevent recurrence:
```bash
0 2 * * 0 docker system prune -af >> /var/log/docker-cleanup.log 2>&1
0 3 * * * docker image prune -af --filter "until=48h" >> /var/log/docker-cleanup.log 2>&1
```

---

## STEP 11: Set up staging environment

What this does:
Staging is a complete copy of production on a separate VM. Code is reviewed on staging
before being merged to main and going to production. Same docker-compose.yml is used
on both VMs — only .env differs.

Standard practice for staging database: empty with fresh migrations.
Staging should never contain real user data.

On staging VM:
```bash
# Install Docker (same commands as production VM)
# Clone the repo
sudo mkdir -p /srv/portfoliosite
sudo chown $USER:$USER /srv/portfoliosite
cd /srv/portfoliosite
git clone https://github.com/Kaushal2059/My_Portfolio_website.git
cd My_Portfolio_website/portfolio

# Create .env with different credentials from production
nano .env
```

Staging .env must have:
```bash
PORTFOLIO_IMAGE=iamkaushal20/portfolio:dev-latest
# Different DB credentials from production
DB_NAME=portfolio_staging
DB_USER=staging_user
DB_PASSWORD=different-password-from-production
POSTGRES_DB=portfolio_staging
POSTGRES_USER=staging_user
POSTGRES_PASSWORD=different-password-from-production
# Use console email backend — no real emails sent from staging
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
```

First deploy on staging:
```bash
docker login -u iamkaushal20
docker compose pull
docker compose up -d
docker compose exec web python manage.py createsuperuser
```

---

## STEP 12: Set up disk maintenance on both VMs

What this does:
Docker accumulates old images, container logs grow indefinitely, and deploy logs
grow forever. These automated jobs keep the disk healthy without manual intervention.

On both VMs — add to crontab:
```bash
crontab -e
```

Add these lines:
```
# Deploy every 5 minutes
*/5 * * * * /srv/portfoliosite/portfolio/deploy.sh

# Remove Docker images older than 2 days daily at 3am
0 3 * * * docker image prune -af --filter "until=48h" >> /var/log/docker-cleanup.log 2>&1

# Full Docker cleanup every Sunday at 2am
0 2 * * 0 docker system prune -af >> /var/log/docker-cleanup.log 2>&1
```

Add logging limits to docker-compose.yml for all services:
```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
# Caps each container log at 30MB total (10MB x 3 files)
```

Set up log rotation on both VMs:
```bash
sudo nano /etc/logrotate.d/portfolio
```

Content:
```
/var/log/portfolio-deploy.log
/var/log/docker-cleanup.log
{
    su kaushal20 kaushal20
    weekly
    rotate 4
    compress
    delaycompress
    missingok
    notifempty
    create 0644 kaushal20 kaushal20
}
```

Verify logrotate config:
```bash
sudo logrotate --debug /etc/logrotate.d/portfolio
```

====>> ERRORS FACED:

Error: logrotate skipping files — insecure permissions

Output:
```
error: skipping "/var/log/portfolio-deploy.log" because parent directory
has insecure permissions. Set "su" directive in config file.
```

Fix: Added `su kaushal20 kaushal20` directive to the logrotate config block.
This tells logrotate which user to run as when rotating files.

---

## FINAL RESULT: Complete pipeline

```
Daily development workflow:

1. Write code on Windows in VS Code
2. git push origin dev
3. GitHub Actions runs automatically:
   - Tests run (SQLite in CI)
   - If tests pass: build :dev-latest image
   - Push to Docker Hub
4. Staging VM picks up new image within 5 minutes via cron
5. Review site at http://STAGING_IP
6. Everything looks good — merge to main:
   git checkout main && git merge dev && git push origin main
7. GitHub Actions runs automatically:
   - Tests run again
   - If tests pass: build :latest image
   - Push to Docker Hub
8. Production VM picks up new image within 5 minutes via cron
9. Live at http://192.168.221.142
```

Key commands for day-to-day use:
```bash
# Check pipeline status
# Go to: github.com/Kaushal2059/My_Portfolio_website/actions

# Check what image is running on VM
docker inspect portfolio-web-1 --format='{{.Config.Image}}'

# Manual deploy without waiting for cron
./deploy.sh

# Watch deploy logs
tail -f /var/log/portfolio-deploy.log

# Check all containers are healthy
docker compose ps

# Roll back to a previous version
# Edit .env: PORTFOLIO_IMAGE=iamkaushal20/portfolio:PREVIOUS_SHA
docker compose up -d --no-deps web
```

---

## Key lessons learned

1. YAML is silently unforgiving — -main and - main look almost identical but one breaks
   the entire workflow with no error message. Always validate YAML syntax before pushing.

2. GitHub Actions only triggers on branches where the workflow file exists on the default
   branch first. Merge the workflow file to main before expecting it to run on dev.

3. Tests passing locally does not mean they pass in CI. A .env file on Windows was
   overriding environment variables set in PowerShell — causing PostgreSQL connection
   attempts in CI even though TEST_DB=sqlite was set. The .env file belongs only on the VM.

4. Push one file at a time and verify before the next. When everything was pushed at once
   and something broke, it was impossible to know which file caused it. Incremental pushes
   made debugging trivial.

5. 9.8GB is too small for a Docker setup running 5+ containers. Docker images accumulate
   quickly. Set up automated cleanup cron jobs from the start and plan for at least 30GB
   for a learning environment with monitoring.

6. Private VM IPs cannot be reached from GitHub Actions. The pull-based deployment pattern
   (VM polls Docker Hub) is a valid real-world solution for air-gapped environments.

7. Read the logs before assuming the tool is broken. Every error in this phase had
   a clear message in the logs pointing to exactly what was wrong.
