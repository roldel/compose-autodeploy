# compose-autodeploy

A minimalist, event-driven auto-deployment system for Docker Compose projects

## Overview

This repository contains a set of scripts and a simple webhook example to automate deployments for Docker Compose-based projects. The workflow is:

1. **Webhook Trigger:** A web framework-based webhook listens for GitHub push events on the main branch and writes a trigger file to a shared volume.
2. **Inotify Watcher:** A host-side script monitors the shared folder using inotify.
3. **Deployment Execution:** When the trigger file is detected, a deployment script pulls the latest code, rebuilds images if necessary, restarts the containers, and prunes unused images.

## Scripts Summary

- **deploy.sh**  
  - **Location:** `/path/to/app/deploy.sh`  
  - **Function:** Pulls code from Git, stops containers, rebuilds images (if Docker-related files have changed), restarts containers, and prunes old images.

- **watch_deploy.sh**  
  - **Location:** `/path/to/watch_deploy.sh`  
  - **Function:** Monitors the shared directory (e.g., `/path/to/shared`) for the trigger file (`deploy_trigger.txt`) using inotifywait. On detection, it runs `deploy.sh` and then removes the trigger file.

- **Webhook Endpoint (views.py)**  
  - **Location:** `webhook/webhook_example/views.py`  
  - **Function:** Listens for GitHub push events, verifies the event (e.g., on the main branch), and writes a trigger file to `/shared/deploy_trigger.txt`.  
  - *(Note: The webhook can be implemented with any web framework, such as Django, Flask, or Laravel.)*

## Setup Summary

1. **Shared Volume:**  
   Create a host directory (e.g., `/path/to/shared`) and mount it into your container at `/shared` via Docker Compose.
   
2. **Install inotify-tools on Alpine:**  
   ```sh
   apk add inotify-tools
    ```
   
3. **OpenRC Service:**  
   Set up an OpenRC service to automatically run watch_deploy.sh on reboot.

## Usage

  1. Webhook:
    Configure your GitHub webhook to point to your webhook endpoint (using application/json as the content type, and selecting just the push event).

  2. Start the Watcher:
    Run the watch_deploy.sh script (or configure it as an OpenRC service).

  3. Trigger Deployment:
    A push to the main branch creates the trigger file, which the watcher detects, then runs the deploy script.

