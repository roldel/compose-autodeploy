# compose-autodeploy

A minimalist, event-driven auto-deployment system for Docker Compose projects.

## Overview

This repository contains scripts and a simple webhook example to automate deployments for Docker Compose-based projects. The workflow is as follows:

1. **Webhook Trigger:** A web framework-based endpoint (e.g., Django, Flask, Node or Laravel) listens for GitHub push events on the main branch and writes a trigger file to a shared volume.
2. **Inotify Watcher:** A host-side script monitors a shared folder using inotify.
3. **Deployment Execution:** When the trigger file is detected, a deployment script pulls the latest code, rebuilds images if needed, restarts the containers, and prunes unused images.

This architecture leverages the existing application server to receive webhook signals without requiring extra dependencies or an additional server. Because the redeploy signal is processed within a container—where executing host-level shell commands violates containerization principles—we use an event-driven trigger file with inotify to propagate the signal to the host. This method ensures minimal resource consumption while keeping the redeployment system perpetually ready.

**Note:** This demonstration uses Alpine Linux as the host operating system. If you are using a different Linux distribution, you may need to make minor adjustments—for example, replacing `apk` with `apt` in shell commands and using `systemctl` instead of OpenRC for service management.


## Scripts Summary

- **deploy.sh**  
  - **Location:** `/path/to/app/deploy.sh`  
  - **Function:** Pulls code from Git, stops containers, rebuilds images (if Docker-related files have changed), restarts containers, and prunes old images.

- **watch_deploy.sh**  
  - **Location:** `/path/to/watch_deploy.sh`  
  - **Function:** Monitors the shared directory (e.g., `/path/to/shared`) for the trigger file (`deploy_trigger.txt`) using inotifywait. When detected, it executes `deploy.sh` and then removes the trigger file.

- **Webhook Endpoint (views.py)**  
  - **Location:** `webhook/webhook_example/views.py`  
  - **Function:** Listens for GitHub push events on the main branch and writes a trigger file to `/shared/deploy_trigger.txt`.  
  - *(Note: The webhook can be implemented with any web framework.)*

## Setup & Usage

1. **Deploy the Webhook Endpoint:**  
   - Set up and deploy your webhook endpoint (e.g., with Django). The endpoint should process GitHub push events, verify that the push is to the main branch, and then write a trigger file (e.g., `/shared/deploy_trigger.txt`) to a shared volume.

2. **Configure the GitHub Webhook:**  
   - In your GitHub repository settings, create a new webhook with:
     - **Payload URL:** The URL of your deployed webhook endpoint (e.g., `https://yourdomain.com/webhook/`).
     - **Content Type:** `application/json`
     - **Secret:** *(Optional – can be added later for security.)*
     - **Events:** Select **Just the push event.**

3. **Set Up the Shared Volume:**  
   - On your host, create a directory (e.g., `/path/to/shared`) and mount it into your container at `/shared` via Docker Compose.


4. **Start the Inotify Watcher:**

    Run the watch_deploy.sh script manually or configure it as an OpenRC service for automatic startup:

    /path/to/watch_deploy.sh

    The watcher monitors the shared directory, and when it detects the trigger file, it runs the deploy script.

5. **Trigger Deployment:**

    A push to the main branch in GitHub sends a webhook to your endpoint, which creates the trigger file. The watcher then detects the file and executes the deployment process.

   
6. **Persist setup on reboot**  
   Set up an OpenRC service to automatically run watch_deploy.sh on reboot.

