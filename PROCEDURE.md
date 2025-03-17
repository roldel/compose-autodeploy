# PROCEDURE

<br>

## Table of content :
<br>

- [STEP 1 : Create a Shared Volume on the Host](#step-1--create-a-shared-volume-on-the-host)

- [STEP 2 : Implement the Django Webhook Endpoint with Secret Verification](#step-2--implement-the-django-webhook-endpoint-with-secret-verification)

- [STEP 3 : Create the GitHub Webhook](#step-3--create-the-github-webhook)

- [STEP 4: Create the Deployment Script on the Host](#step-4--create-the-deployment-script-on-the-host)

- [STEP 5: Create the inotify Monitoring Script on the Host](#step-5--create-the-inotify-monitoring-script-on-the-host)

- [STEP 6: Create an OpenRC Service for the Watcher](#step-6--create-an-openrc-service-for-the-watcher)

- [STEP 7: Final Summary and Workflow Overview](#step-7--final-summary-and-workflow-overview)


<br>


## STEP 1 : Create a Shared Volume on the Host

### On the Host:

Create a directory that will be shared with your container (replace \<USERNAME> with your actual username):

```sh
mkdir -p /home/<USERNAME>/shared
```

### Docker Compose Mount:

In your docker-compose.yml, mount the host directory into your container so that any file written to the shared folder is visible both inside the container and on the host.

```yml
services:
  web:
    image: your-django-image
    environment:
      - GITHUB_WEBHOOK_SECRET=your-strong-secret-here
    volumes:
      - /home/<USERNAME>/shared:/shared  # Maps host directory to /shared in the container
    ports:
      - "8000:8000"
    # ... other configurations ...
```

Inside the container, files written to /shared will be stored on the host at /home/\<USERNAME>/shared.  


<br>

---

<br>
<br>

## STEP 2 : Implement the Django Webhook Endpoint with Secret Verification

### In your Django app (for example, in views.py), add an endpoint that:

- Verifies the HMAC signature using the secret
- Checks for a push event on the main branch
- Writes a trigger file to the shared volume

<br>

```python
import json
import hmac
import hashlib
import os
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt

# Retrieve the webhook secret from the environment
WEBHOOK_SECRET = os.environ.get("GITHUB_WEBHOOK_SECRET", "defaultsecret")
# Path in the shared volume – must match the mount point in the container.
SHARED_TRIGGER_FILE = '/shared/deploy_trigger.txt'

def verify_signature(payload, signature):
    """
    Verify the payload signature using HMAC SHA256.
    GitHub sends the signature in the header 'X-Hub-Signature-256' in the format 'sha256=...'
    """
    if not signature:
        return False

    mac = hmac.new(WEBHOOK_SECRET.encode(), msg=payload, digestmod=hashlib.sha256)
    expected_signature = f"sha256={mac.hexdigest()}"
    return hmac.compare_digest(expected_signature, signature)

@csrf_exempt
def github_webhook(request):
    if request.method != 'POST':
        return JsonResponse({'message': 'Invalid HTTP method'}, status=405)

    # Get signature from header
    signature = request.META.get('HTTP_X_HUB_SIGNATURE_256', '')
    payload = request.body

    # Verify the signature
    if not verify_signature(payload, signature):
        return JsonResponse({'error': 'Invalid signature'}, status=403)

    try:
        data = json.loads(payload)
    except Exception:
        return JsonResponse({'error': 'Invalid JSON'}, status=400)

    # Check for the GitHub event type header
    event_type = request.META.get('HTTP_X_GITHUB_EVENT', '')
    if event_type != 'push':
        return JsonResponse({'message': 'Not a push event'}, status=200)

    # Ensure the push is on the main branch
    branch_ref = data.get('ref', '')
    if branch_ref == 'refs/heads/main':
        try:
            with open(SHARED_TRIGGER_FILE, 'w') as f:
                f.write('deploy')
            return JsonResponse({'message': 'Deployment triggered'}, status=200)
        except Exception:
            return JsonResponse({'error': 'Could not create trigger file'}, status=500)
    else:
        return JsonResponse({'message': 'Push event to non-main branch ignored'}, status=200)
```

Configure the URL for this view (for example, /webhook/) in your Django URL configuration.

<br>

---

<br>

## STEP 3 : Create the GitHub Webhook

### Navigate to Your Repository:

In GitHub, go to Settings → Webhooks → Add webhook.

### Configure the Webhook:

**Payload URL:** Set this to the public URL where your Django app is reachable (e.g., https://<YOUR_DOMAIN>/webhook/)  

**Content Type:** Select application/json  

**Secret:** Enter the same secret as in your Django container (e.g., your-strong-secret-here)  

**SSL Verification:** Enable SSL verification (recommended if you have a valid certificate)  

**Which events would you like to trigger this webhook?** : Select "Just the push event."

**Save the Webhook:** Click "Add webhook" to complete the setup

<br>

---

<br>

## STEP 4 : Create the Deployment Script on the Host

Place the following deployment script on the host (e.g., at /home/\<USERNAME>/\<PROJECT>/deploy.sh). This script pulls the latest code, rebuilds images if necessary, stops and restarts containers, and cleans up old images. Use /bin/sh for Alpine compatibility.

```sh
#!/bin/sh

APP_DIR="/home/<USERNAME>/<PROJECT>"
GIT_REPO="origin/main"
DOCKER_COMPOSE_FILE="$APP_DIR/docker-compose.yml"

echo "Starting deployment..."

# Step 1: Pull latest code
cd "$APP_DIR" || exit 1
echo "Fetching latest changes..."
git pull $GIT_REPO || { echo "Git pull failed! Deployment aborted."; exit 1; }

# Step 2: Stop running containers
echo "Stopping running containers..."
docker compose down || { echo "Failed to stop containers! Deployment aborted."; exit 1; }

# Step 3: Build new images (if necessary)
if git diff --name-only HEAD^ HEAD | grep -E "(.*/Dockerfile|docker-compose.yml|.*/requirements.txt)"; then
    echo "Changes detected in Docker-related files. Rebuilding images..."
    docker compose build --pull || { echo "Docker build failed! Deployment aborted."; exit 1; }
fi

# Step 4: Start new containers
echo "Starting new containers..."
docker compose up -d --force-recreate || { echo "Restart failed! Deployment aborted."; exit 1; }

# Step 5: Clean up old images
echo "Removing old, unused images..."
docker image prune -f

echo "Deployment complete!"
exit 0
```


Make the script executable:

```sh
chmod +x /home/<USERNAME>/<PROJECT>/deploy.sh
```

<br>

---

<br>

## STEP 5 : Create the inotify Monitoring Script on the Host

Create a script (for example, /home/\<USERNAME>/watch_deploy.sh) that monitors the shared folder for the trigger file. Again, use /bin/sh:

```sh
#!/bin/sh

# Configuration - Replace placeholders with actual values
WATCH_DIR="/home/<USERNAME>/shared"
TRIGGER_FILE="$WATCH_DIR/deploy_trigger.txt"
DEPLOY_SCRIPT="/home/<USERNAME>/<PROJECT>/deploy.sh"
LOG_FILE="/var/log/deploy_watch.log"
LOCK_FILE="/tmp/deploy.lock"

# Function for timestamped logging
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Ensure log file exists and is writable BEFORE writing to it
touch "$LOG_FILE" && chmod 664 "$LOG_FILE"

# Ensure inotifywait is installed before proceeding
command -v inotifywait >/dev/null 2>&1 || { log "ERROR: inotifywait not found! Install inotify-tools."; exit 1; }

log "Monitoring $WATCH_DIR for deployment trigger..."

# Use inotifywait in monitor mode
inotifywait -m -e create -e modify "$WATCH_DIR" | while read path action file; do
    if [ "$file" = "deploy_trigger.txt" ]; then
        log "Trigger file detected: $file ($action)"

        # Remove trigger file immediately to ensure new changes aren't ignored
        [ -f "$TRIGGER_FILE" ] && rm -f "$TRIGGER_FILE"

        # Ensure that deployment requests queue up
        while [ -f "$LOCK_FILE" ]; do
            log "Deployment already in progress, waiting..."
            sleep 5  # Wait and check again
        done

        touch "$LOCK_FILE"

        # Execute deployment script
        if "$DEPLOY_SCRIPT" >> "$LOG_FILE" 2>&1; then
            log "Deployment executed successfully."
        else
            log "ERROR: Deployment failed!"
        fi

        # Cleanup
        rm -f "$LOCK_FILE"
    fi
done


```

Make this script executable:

```sh
chmod +x /home/<USERNAME>/watch_deploy.sh
```

Ensure inotify-tools is installed on your Alpine host:

```sh
apk add inotify-tools
```

<br>

---

<br>

## STEP 6 : Create an OpenRC Service for the Watcher

To ensure your watcher script starts automatically on reboot, create an OpenRC init script.

Create the Service File:

- Create a file at /etc/init.d/webhook-watcher with the following contents:

```sh
#!/sbin/openrc-run
description="Webhook Deployment Watcher Service"
command="/home/<USERNAME>/watch_deploy.sh"
command_background=true
pidfile="/var/run/webhook-watcher.pid"
name="webhook-watcher"

depend() {
    need net
}
```

- Make the Service Executable:

```sh
chmod +x /etc/init.d/webhook-watcher
```

- Enable and Start the Service:

```sh
rc-update add webhook-watcher default
rc-service webhook-watcher start
```


<br>

---

<br>

## STEP 7 : Final Summary and Workflow Overview

    Shared Volume Setup:
        Host: Create /home/<USERNAME>/shared.
        Docker Compose: Mount it into the container at /shared.

    Django Webhook Endpoint:
        Implements secret verification using GITHUB_WEBHOOK_SECRET.
        Listens at, for example, /webhook/.
        Writes a trigger file (deploy_trigger.txt) to /shared on a valid push event to the main branch.

    GitHub Webhook Configuration:
        Payload URL: https://<YOUR_DOMAIN>/webhook/
        Content Type: application/json
        Secret: Set to the same value as GITHUB_WEBHOOK_SECRET.
        Events: Just the push event.
        SSL Verification: Enabled (if you have a valid certificate).

    Deployment Script:
        Located at /home/<USERNAME>/<PROJECT>/deploy.sh.
        Pulls updates, rebuilds images if necessary, and restarts Docker containers.

    inotify Watcher Script:
        Located at /home/<USERNAME>/watch_deploy.sh.
        Monitors /home/<USERNAME>/shared for the trigger file.
        Executes the deployment script when the file is detected, then removes the trigger file.

    OpenRC Service:
        The service /etc/init.d/webhook-watcher ensures that the watcher script starts automatically on reboot and remains running.

Replace \<USERNAME>, \<PROJECT>, and \<YOUR_DOMAIN> with your actual values. This complete procedure provides an automated, secure, event-driven deployment process triggered by GitHub webhooks.