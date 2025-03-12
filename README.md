# Compose-AutoDeploy

**A Minimalist, Event-Driven Auto-Deployment System for Docker Compose Projects**

## Overview

This repository provides a lightweight, event-driven solution for automating deployments of Docker Compose-based projects. The system is designed to streamline the deployment process by leveraging webhooks and file-based triggers, ensuring seamless updates with minimal resource overhead.

### Key Features:
- **Webhook Integration**: A web framework-based endpoint (e.g., Django, Flask, Node.js, or Laravel) listens for GitHub push events on the `main` branch. Upon detecting a push, it writes a trigger file to a shared volume.
- **Inotify Monitoring**: A host-side script monitors the shared volume using `inotify`, ensuring real-time detection of the trigger file.
- **Automated Deployment**: When the trigger file is detected, a deployment script executes the following steps:
  - Pulls the latest code from the repository.
  - Rebuilds Docker images if necessary.
  - Restarts the containers.
  - Prunes unused Docker images to maintain a clean environment.

### Architecture Highlights:
- **Containerized Workflow**: The system adheres to containerization principles by decoupling the webhook listener (running in a container) from the host-level deployment tasks. This is achieved through an event-driven trigger file mechanism.
- **Resource Efficiency**: By utilizing `inotify` for file monitoring, the system remains perpetually ready without consuming significant resources.
- **Flexibility**: The solution is designed to work with any web framework capable of handling webhooks, making it adaptable to diverse project requirements.

### Host Compatibility:
This demonstration is optimized for **Alpine Linux** as the host operating system. For other Linux distributions, minor adjustments may be required, such as:
- Replacing `apk` with `apt` for package management.
- Using `systemctl` instead of OpenRC for service management.
