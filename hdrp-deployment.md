## Automated Application Deployment to VM using GitHub Actions (with Rollback Support)

### Concept Overview

This project automates the deployment of a web application to a virtual machine (VM) using **GitHub Actions**. The solution:
- Minimizes manual intervention and deployment errors.
- Implements a rollback mechanism to revert to the last stable state in case of failure.
- Ensures high availability and faster recovery, especially for production environments.

### Objective
To implement a robust and reliable CI/CD pipeline using:
- GitHub Actions for automation,
- Rsync for efficient file transfers,
- Systemd-managed services like Gunicorn and Celery,
- Rollback support in case of deployment issues,
- Conditional service restarts based on code changes.

### Architecture Overview
```text
Developer Pushes Code → GitHub Actions Workflow (.github/workflows/deploy.yml)
                         ↓
                  Self-Hosted Runner on VM (EC2)
                         ↓
                Bash Scripts: deploy.sh / rollback_latest.sh
                         ↓
        Gunicorn, Nginx, Celery, and Celery Beat Restarted as Needed
                         ↓
            Slack Notifications for Success or Rollback

```

### Deployment Workflow
1. Developer pushes code to main or master.
2. GitHub Actions picks up the push via self-hosted runner.
3. deploy.sh is triggered by workflow and performs:
     - Create timestamped directory in /home/ubuntu/releases/<timestamp>
     - Copy source code from /home/ubuntu/git-source (not directly from .git)
     - Set up or reuse shared Python virtual environment
     - Install dependencies if requirements.txt changed
     - Apply Django database migrations
     - Copy .env file and link latest release with: `ln -sfn /home/ubuntu/releases/<timestamp> /home/ubuntu/current`
     - Restart services (Gunicorn, Celery, etc.)
4. If deployment fails:
     - rollback_latest.sh is triggered
     - Restores symlink to the previous release and Restarts services to revert to stable state
5. Slack Notifications:
     - ✅ Success → "Deployment Successful"
     - ❌ Failure → "Deployment Failed - Rollback Triggered"

## Repository Structure
```
Project Structure:

├── .github/
│   └── workflows/
│       └── deploy.yml             # GitHub Actions CI/CD pipeline
├── /srv/hdrp-staging/                # Directory holding the checked-out source code
├── releases/                     # Timestamped releases created by deploy.sh
│   └── <timestamped-release>/    # Actual release directories
│       └── current/    # Django project within each release
├── run/deploy.sh                     # Main deployment script
├── run/rollback_latest.sh            # Script to revert to the previous stable release
├── README.md                     # Documentation and runbook

```

### Automated deployment process
##### Scripts - Create an automated deployment and rollback scripts.
***deploy.sh***: To automate the application deployment process with zero downtime.
Location: deploy.sh: `/srv/run/deploy.sh`
```script.sh

#!/bin/bash

set -euo pipefail

BRANCH="${1:-master}"

APP_DIR="/srv/hdrp-staging"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RELEASE_NAME="release_$TIMESTAMP"
RELEASES_DIR="$APP_DIR/releases"
RELEASE_DIR="$APP_DIR/releases/$RELEASE_NAME"
SHARED_VENV="$APP_DIR/shared/venv" 
REQ_HASH_FILE="$SHARED_VENV/.requirements_hash"


# Function for logging
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Create new release directory
log "Creating new release directory: $RELEASE_NAME"
mkdir -p "$RELEASES_DIR"


log "Starting deployment for branch '$BRANCH'"
# Pull latest code
log "Pulling latest code from Git..."
cd "$APP_DIR"

log "Cloning fresh code from Git (no cache)..."
git clone \
    --branch "$BRANCH" \
    --depth 1 \
    --no-tags \
    https://github.com/Hiring-Dog/HDRP-Backend.git \
    "$RELEASE_DIR"

cp "$APP_DIR/shared/staging.py" "$RELEASE_DIR/hdrpbackend/settings/staging.py"


# ===== Set up Python Environment =====
log "Setting up Python environment..."
if [ ! -d "$SHARED_VENV" ]; then
    log "Creating shared virtual environment at $SHARED_VENV"
    python3 -m venv "$SHARED_VENV"
fi

#log "Linking shared virtual environment..."
#ln -sfn "$SHARED_VENV" "$RELEASE_DIR/venv"
log "Activating virtual environment..."
source "$SHARED_VENV/bin/activate"

log "Checking dependencies..."
REQ_HASH=$(sha256sum $RELEASE_DIR/requirements.txt 2>/dev/null | awk '{print $1}' || echo "")
if [ ! -f "$REQ_HASH_FILE" ] || [ "$(cat "$REQ_HASH_FILE" 2>/dev/null)" != "$REQ_HASH" ]; then
    log "$RELEASE_DIR/requirements.txt changed. Installing dependencies..."
    pip install --upgrade pip
    pip install --no-cache-dir -r $RELEASE_DIR/requirements.txt
    echo "$REQ_HASH" > "$REQ_HASH_FILE"
else
    log "No changes in requirements.txt. Skipping pip install."
fi

# ===== Environment and Django checks =====
log "Checking staging.env file..."
if [ ! -f "$APP_DIR/shared/env/staging.env" ]; then
    log "[WARNING] staging.env file not found. Copying from template if exists..."
fi


# ===== Run Migrations and Health Checks =====
cd "$RELEASE_DIR"
log "Running migrations..."

#set -a && source /srv/hdrp-staging/shared/env/staging.env && set +a
python manage.py migrate --noinput

log "Performing Django health check..."
if ! python manage.py check; then
    log "[ERROR] Django check failed. Deployment aborted."
    exit 1
fi

log "Checking for unapplied migrations..."
if ! python manage.py makemigrations --check --dry-run; then
    log "[WARNING] Uncommitted model changes detected."
fi

# update 'current' symlink in releases directory
ln -sfn "$RELEASE_DIR" "$APP_DIR/current"
log "updated 'current' symlink: $APP_DIR/current -> $RELEASE_DIR"

# ===== Get Commit Information =====
log "Getting commit information for restart decisions..."

# Temporarily move into the application repo directory to run Git commands
pushd "$RELEASE_DIR" > /dev/null

CURRENT_COMMIT=$(git rev-parse HEAD)
# Last commit message for the commit being deployed
COMMIT_MSG=$(git log -1 --pretty=%B "$CURRENT_COMMIT" 2>/dev/null || echo "")

# Read the last deployed Git commit hash from a file (if it exists).
LAST_DEPLOYED_COMMIT=$(cat "$APP_DIR/.last_deployed_commit" 2>/dev/null || echo "")

# If we have a last deployed commit, compare it to the current commit and get a list of files that changed.
if [ -n "$LAST_DEPLOYED_COMMIT" ]; then
    CHANGED_FILES=$(git diff --name-only "$LAST_DEPLOYED_COMMIT" "$CURRENT_COMMIT" 2>/dev/null || echo "")
else
    CHANGED_FILES="initial"  # fallback
fi

# Files/patterns that typically affect Celery behavior
CELERY_AFFECTING_REGEX='(tasks\.py|celery(\.py|/)|celeryconfig\.py|hdrpbackend/celery\.py|requirements\.txt|pyproject\.toml|poetry\.lock|settings\.py|hdrpbackend/settings/.*\.py|.*models\.py|.*utils\.py|services/|jobs/)'

SHOULD_RESTART=false

# File-based detection (most cases)
if echo "$CHANGED_FILES" | grep -Eiq "$CELERY_AFFECTING_REGEX"; then
    SHOULD_RESTART=true
elif echo "$COMMIT_MSG" | grep -Eiq '\[(restart-?celery|restart:celery|restart:all)\]'; then
    SHOULD_RESTART=true
fi
# Return to previous directory
popd > /dev/null

log "Current commit: $CURRENT_COMMIT"
log "Commit message: $COMMIT_MSG"
log "Changed files: $(echo "$CHANGED_FILES" | wc -l) files"
log "Should restart Celery: $SHOULD_RESTART"



RESTART_RABBITMQ=false
RESTART_REDIS=false
# ===== Restart Services =====
log "Reloading/restarting services..."

if echo "$COMMIT_MSG" | grep -Eiq '\[(restart-?gunicorn|restart:gunicorn|restart:all)\]'; then
    log "Commit message contains restart keyword. Performing full restart..."
    sudo systemctl daemon-reload || true
    sudo systemctl restart gunicorn
    sleep 3
    systemctl is-active --quiet gunicorn|| { log "[ERROR] Gunicorn failed to restart"; exit 1; }
    log "Gunicorn restarted successfully."
else
    log "No restart keyword found. Performing graceful reload (zero-downtime)..."
    

    # Check if Gunicorn is running
    if systemctl is-active --quiet gunicorn; then
        log "Performing zero-downtime Gunicorn reload..."
    # Get the master process PID
        GUNICORN_PID=$(systemctl show --property MainPID gunicorn | cut -d= -f2)
    
        if [[ "$GUNICORN_PID" != "0" ]]; then
        # Send USR2 to start new master with new workers
            sudo kill -USR2 "$GUNICORN_PID"
        
        # Wait for new master to start
            sleep 5
         # Check if new PID file was created
            if [[ -f /run/gunicorn/gunicorn.pid.2 ]]; then
                NEW_PID=$(cat /run/gunicorn/gunicorn.pid.2)
                echo "$NEW_PID" | sudo tee /run/gunicorn/gunicorn.pid
                log "Updated PID file to new master: $NEW_PID"
        
        # Send WINCH to old master to gracefully shut down old workers
                sudo kill -WINCH "$GUNICORN_PID"
        
        # Wait a bit more for graceful shutdown
                sleep 5
        
        # Send TERM to old master to shut it down completely
                sudo kill -TERM "$GUNICORN_PID"
        
                log "Zero-downtime Gunicorn reload completed"
            else
                log "[WARNING] No new master spawned, falling back to restart"
                sudo systemctl restart gunicorn
            fi    
        else
            log "Could not get Gunicorn PID, falling back to systemctl restart"
            sudo systemctl restart gunicorn
        fi
    else
    log "Starting Gunicorn..."
    sudo systemctl start gunicorn
    fi
fi


###### Graceful restart of Celery services #####
log "Checking if Celery restart is needed..."


if [ "$SHOULD_RESTART" = true ]; then
    log "Restarting Celery and Celery Beat..."
    sudo systemctl restart celery
    sudo systemctl restart celery-beat
    sleep 5
    systemctl is-active --quiet celery || { log "[ERROR] Celery failed to restart"; exit 1; }
    systemctl is-active --quiet celery-beat || { log "[ERROR] Celery Beat failed to restart"; exit 1; }
    log "Celery services restarted successfully"
else
    log "No changes affecting Celery. Skipping restart."
fi

# ===== RabbitMQ Restart Logic =====
if echo "$COMMIT_MSG" | grep -Eiq '\[(restart-?rabbitmq|restart:rabbitmq|restart:all)\]'; then
    RESTART_RABBITMQ=true
    log "RabbitMQ restart requested via commit message"
    sudo systemctl restart rabbitmq-server
    sleep 10
    systemctl is-active --quiet rabbitmq-server || { log "[ERROR] RabbitMQ failed to restart"; exit 1; }
    log "RabbitMQ restarted successfully"
else
    log "RabbitMQ restart not needed"
fi

# ===== Redis Restart Logic =====
if echo "$COMMIT_MSG" | grep -Eiq '\[(restart-?redis|restart:redis|restart:all)\]'; then
    RESTART_REDIS=true
    log "Redis restart requested via commit message"
elif echo "$CHANGED_FILES" | grep -Eiq '(redis|cache|session|celery.*backend)'; then
    RESTART_REDIS=true
    log "Redis restart needed due to cache/session configuration changes"
else
    log "Redis restart not needed"
fi

# RabbitMQ restart
if [ "$RESTART_RABBITMQ" = true ]; then
    log "Restarting RabbitMQ..."
    sudo systemctl restart rabbitmq-server
    sleep 10
    systemctl is-active --quiet rabbitmq-server || { log "[ERROR] RabbitMQ failed to restart"; exit 1; }
    log "RabbitMQ restarted successfully"
else
    log "RabbitMQ not restarted (no relevant changes)"
fi

# Redis restart
if [ "$RESTART_REDIS" = true ]; then
    log "Restarting Redis..."
    sudo systemctl restart redis-server
    sleep 3
    systemctl is-active --quiet redis-server || { log "[ERROR] Redis failed to restart"; exit 1; }
    log "Redis restarted successfully"
else
    log "Redis not restarted (no relevant changes)"
fi

# ===== Save Deployment State =====

echo "$CURRENT_COMMIT" > "$APP_DIR/.last_deployed_commit"
log "[SUCCESS] Deployment complete: $RELEASE_DIR"

# ===== Clean Up Old Releases =====
log "Cleaning up old releases (keeping last 5)..."
cd "$RELEASES_DIR"
ls -1dt "$RELEASES_DIR"/release_* | tail -n +6 | xargs -r rm -rf


exit 0
```

#### rollback_latest.sh - Reverts the symlink to the previous stable release if something fails.
location: : /run/run/rollback_latest.sh
```
#!/bin/bash
set -euo pipefail

log() {
    echo "[`date +%Y-%m-%d\ %H:%M:%S`] $1"
}
APP_DIR="/srv/hdrp-staging"
RELEASES_DIR="$APP_DIR/releases"
SYMLINK="/srv/hdrp-staging/current"

RELEASES=($(ls -td ${RELEASES_DIR}/*))

if [ ${#RELEASES[@]} -lt 2 ]; then
    log "[ERROR] Not enough releases to perform rollback."
    exit 1
fi

CURRENT=${RELEASES[0]}
PREVIOUS=${RELEASES[1]}

if [ ! -d "$PREVIOUS" ]; then
    log "[ERROR] Previous release directory does not exist: $PREVIOUS"
    exit 1
fi

log "Current symlink points to: $(readlink -f $SYMLINK)"
log "Rolling back to previous release: $PREVIOUS"

# Update the symlink atomically
ln -sfn "$PREVIOUS" "$SYMLINK"

# Restart services
log "Restarting Celery services..."
if ! sudo systemctl restart celery; then
    log "[ERROR] Failed to restart Celery."
    exit 1
fi
if ! sudo systemctl restart celery-beat; then
    log "[ERROR] Failed to restart Celery Beat."
    exit 1
fi

log "Reloading/restarting Gunicorn..."
if systemctl is-active --quiet gunicorn; then
    log "Performing zero-downtime Gunicorn restart..."
    
    # Get the master process PID
    GUNICORN_PID=$(systemctl show --property MainPID gunicorn | cut -d= -f2)
    
    if [[ "$GUNICORN_PID" != "0" ]]; then
        # Send USR2 to start new master with new workers
        sudo kill -USR2 "$GUNICORN_PID" 

        # Wait for new master to start
        sleep 3      

        # Send WINCH to old master to gracefully shut down old workers
        sudo kill -WINCH "$GUNICORN_PID"

        # Wait a bit more for graceful shutdown
        sleep 2
        
        # Send TERM to old master to shut it down completely
        sudo kill -TERM "$GUNICORN_PID"
        
        log "Zero-downtime Gunicorn restart completed"
    else
        log "Could not get Gunicorn PID, falling back to systemctl restart"
        sudo systemctl restart gunicorn
    fi
else
    log "Starting Gunicorn..."
    sudo systemctl start gunicorn
fi

log "[SUCCESS] Rollback complete."
log "New symlink points to: $(readlink -f $SYMLINK)"
```

Script Name	|Description	| Path
deploy.sh	  |Handles deployment with zero downtime	|/home/ubuntu/deploy.sh
rollback_latest.sh	|Handles rollback to last stable release	|/home/ubuntu/rollback_latest.sh

### Permissions:
``` bash
chmod +x /srv/hdrp-staging/run/deploy.sh
chmod +x /home/hdrp-staging/run/rollback_latest.sh
```

### Deployment Strategy 

#### 1. Conditional Dependency Installation 
Only install Python dependencies if requirements.txt has changed, saving time and resources. Avoids unnecessary reinstallation, speeds up deploys, and ensures dependencies are always up-to-date when needed.
***Details:***
- Computes a SHA256 hash of requirements.txt.
- Compares it to the last deployed hash (stored in $REQ_HASH_FILE).
```bash
# Only installs if requirements.txt changed
REQ_HASH=$(sha256sum requirements.txt | awk '{print $1}')
if [ ! -f "$REQ_HASH_FILE" ] || [ "$(cat $REQ_HASH_FILE)" != "$REQ_HASH" ]; then
pip install --no-cache-dir -r requirements.txt
fi
```
- If changed or first deploy: 
    - Upgrades pip for latest features and security.
    - Installs dependencies with --no-cache-dir to avoid stale packages.
    - Stores the new hash for future comparison.

#### 2. Timestamp-based releases
We use timestamp-based releases combined with symbolic links to manage and switch deployments cleanly.  
This strategy allows us to:
- Instantly revert to any previous release in seconds
- Keep multiple releases available simultaneously
- Switch to a new release without downtime
```bash
TIMESTAMP=$(date +%Y%m%d%H%M%S)
RELEASE_DIR="$RELEASES_DIR/$TIMESTAMP"
ln -sfn "$RELEASE_DIR" /srv/hdrp-staging/current
```
- **Original Git Repository**:  
  `/home/hdrp-staging/releases/release_timestamp`  
  Contains the cloned GitHub repo (with `.git/`).
- **Timestamped Release Directories**:  
  `/home/hdrp-staging/releases/<timestamp>`  
  Each deployment creates a new folder named by timestamp (e.g., `20250611111202`), created via `rsync` from the Git repo.
- **Active Symlink Directory**:  
  `/home/hdiplatform/Hiringdog-backend`  
  This is a symbolic link that always points to the **currently active release** of the application.
  ```bash
  ln -sfn "$RELEASE_DIR" /home/hdiplatform/Hiringdog-backend
  ```
This ensures minimal disruption and safer, atomic deployments.

#### 3. Zero Downtime Reload (Gunicorn)
Performs a blue/green style, zero-downtime reload of the Gunicorn application server, ensuring no dropped requests during deploys.
Sends signals in sequence:
- USR2: Starts a new master process with new workers (loads new code).
- WINCH: Gracefully shuts down old workers.
- TERM: Terminates the old master process
``` 
  kill -USR2 "$GUNICORN_PID"      # start new master
  kill -WINCH "$GUNICORN_PID"     # gracefully stop old workers
  kill -TERM "$GUNICORN_PID"      # stop old master
```
**Zero Downtime:**
- **No Request Loss**: All requests are handled during reload
- **Seamless Transition**: Users don't experience any interruption


#### 4. Selective Celery Restart
 Restart only if celery.py/, tasks.py/, or related files have changed:
```bash
# Check if Celery-related files changed
if echo "$CHANGED_FILES" | grep -E 'tasks\.py|celery(\.py|/)|requirements\.txt'; then
    sudo systemctl restart celery
    sudo systemctl restart celery-beat
else
    log "No changes affecting Celery. Skipping restart."
fi
```
- **Faster Deployments**: Saves 10-15 seconds when Celery restart isn't needed
- **Reduced Disruption**: Avoids unnecessary task queue interruptions
- **Resource Efficiency**: Prevents unnecessary CPU and memory usage

#### 5. Rollback Strategy
Script: ` rollback_latest.sh `
Rollback Logic:
   - Identify the last successful release (based on timestamp)
   - Point current symlink to that release
   - Restart services to revert to previous state.

## Self-Hosted Runner Setup in a VM
### 1. Create a Dedicated User for GitHub Runner
```bash
sudo adduser --disabled-password --gecos "" github-runner
```
This user is isolated and used only for GitHub Actions. It has no root access by default.

### 2. Register the Runner in GitHub
- Go to your GitHub repo → Settings → Actions → Runners
- Click "New self-hosted runner"
- Choose Linux x64, follow setup instructions.
```
# Install runner
cd /home/github-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Configure runner
./config.sh --url https://github.com/your-org/your-repo --token YOUR_TOKEN
```

### 3. Install the Runner as a Service
```bash
cd /home/github-runner/actions-runner
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

### User Permissions
User - `ubuntu`- Runs the application services
User- `github-runner`- Executes GitHub Actions workflows

Use sudo visudo -f /etc/sudoers.d/github-runner to give NOPASSWD for only required commands.
Example: 
```
github-runner ALL=(ALL) NOPASSWD: /bin/systemctl restart gunicorn, \
                                  /bin/systemctl restart celery, \
                                  /bin/systemctl restart celery-beat, \
                                  /bin/systemctl restart redis-server, \
                                  /bin/systemctl restart rabbitmq-server, \
                                  /bin/systemctl reload gunicorn, \
                                  /bin/systemctl start gunicorn, \
                                  /bin/systemctl stop gunicorn, \
                                  /bin/systemctl is-active *, \
                                  /bin/systemctl status *

# Allow managing GitHub runner services
github-runner ALL=(ALL) NOPASSWD: /home/github-runner/actions-runner/svc.sh, \
                                /bin/systemctl start actions.runner*, \
                                /bin/systemctl stop actions.runner*, \
                                /bin/systemctl status actions.runner*
github-runner ALL=(ubuntu) NOPASSWD: /home/hdiplatform/rollback_latest.sh
github-runner ALL=(ubuntu) NOPASSWD: /home/hdiplatform/deploy.sh
```

#### Create the following workflow file: file_path:`.github/workflows/staging-deploy.yml`
```
name: Deploy to VM via Self-hosted Runner

on:
  push:
    branches: [staging-production]

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run deployment script on VM
        run: |
          sudo -u hdiplatform /home/hdiplatform/deploy.sh

      - name: Rollback if Deployment Fails
        if: failure()
        run: |
          echo "Deployment failed. Rolling back..."
          sudo -u hdiplatform /home/hdiplatform/rollback_latest.sh

      - name: Check service status
        run: |
          systemctl status gunicorn --no-pager
          systemctl status celery --no-pager
          systemctl status celery-beat --no-pager

      - name: ✅ Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": ":white_check_mark: *Deployment Success*\nBranch: `${{ github.ref_name }}`\nTime: `${{ github.event.head_commit.timestamp }}`"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: ❌ Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": ":x: *Deployment Failed*\nBranch: `${{ github.ref_name }}`\nPlease check logs. Rollback has been triggered."
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

