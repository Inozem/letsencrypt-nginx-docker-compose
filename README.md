# Automated Nginx Deployment with Let's Encrypt and WebSocket for Azure Realtime STT

## Overview

This solution is a Docker Compose configuration that automates the deployment of Nginx with SSL certificates via Let's Encrypt and WebSocket setup for integration with Azure Realtime STT (Speech-to-Text). The solution also includes a mechanism for automatic SSL certificate renewal and Nginx reload.

## Components

1. **Nginx** — A web server configured to work with HTTPS and WebSocket support required for Azure Realtime STT.
2. **Let's Encrypt (Certbot)** — A service for automatically generating and renewing SSL certificates.
3. **Docker Compose** — A tool for managing containers, providing an easy way to deploy and manage Nginx, Certbot, and your project.
4. **WebSocket** — Configured to handle Azure Realtime STT requests via the `/socket.io/` route.
