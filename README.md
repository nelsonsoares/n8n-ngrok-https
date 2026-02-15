# n8n with Ngrok Tunnel

This repository contains a Docker Compose setup for running **n8n** with **Ngrok** as a tunneling service. n8n is a workflow automation tool that allows you to connect different services and APIs. Ngrok exposes local servers behind NATs and firewalls to the public internet over secure tunnels.

> Based on [joffcom/n8n-ngrok](https://github.com/joffcom/n8n-ngrok) with additional configuration for HTTPS and OAuth support.

## Prerequisites

Before you begin, ensure you have the following installed:
- [Docker](https://docs.docker.com/get-docker/) or [Podman](https://podman.io/getting-started/installation)
- [Docker Compose](https://docs.docker.com/compose/install/) (or `podman compose`)

## Setup

### 1. Clone the Repository

```bash
git clone https://github.com/nelsonsoares/n8n-ngrok-https.git
cd n8n-ngrok-https
```

### 2. Ngrok Authentication

You need an Ngrok account. If you don't have one, create it at [ngrok.com](https://ngrok.com/). After creating an account, get your auth token from the Ngrok dashboard.

### 3. Reserve a Permanent Domain

In your Ngrok Dashboard, reserve a domain under **Cloud Edge > Domains**. Once you have the domain, proceed to the next step.

### 4. Configure Environment Variables

Create a `.env` file (this file is git-ignored for security):

```sh
TIMEZONE=America/Sao_Paulo
NGROK_TOKEN=your_ngrok_auth_token_here
URL=https://your-domain.ngrok-free.dev
```

> [!IMPORTANT]
> The `URL` variable **must** include the `https://` prefix. Without it, n8n will generate OAuth Redirect URLs without the protocol, which will cause OAuth authentication (e.g., Gmail, Google Sheets) to fail.

### 5. Configure Ngrok Tunnel

Update the `ngrok.yml` file with your reserved domain (without protocol):

```yaml
version: 2
log_level: debug
tunnels:
    n8n:
        proto: http
        addr: n8n:5678
        domain: your-domain.ngrok-free.dev
```

## Key Configuration Details

### OAuth Redirect URL (HTTPS)

The `docker-compose.yaml` includes two critical environment variables:

| Variable | Purpose |
|----------|---------|
| `WEBHOOK_URL` | Sets the base URL for webhooks |
| `N8N_EDITOR_BASE_URL` | Sets the base URL for the n8n editor, ensuring OAuth Redirect URLs are generated with `https://` |

Both use the `URL` variable from `.env`, which **must** start with `https://`. This ensures that when configuring OAuth credentials (e.g., Gmail), the redirect URL will be:

```
https://your-domain.ngrok-free.dev/rest/oauth2-credential/callback
```

This URL is what you need to add as an **Authorized redirect URI** in Google Cloud Console (or any OAuth provider).

## Running the Application

```bash
# With Docker
docker-compose up -d

# With Podman
podman compose up -d
```

## Accessing n8n

After starting the services, access n8n by navigating to your Ngrok domain in your browser:

```
https://your-domain.ngrok-free.dev
```

## Stopping the Application

```bash
# With Docker
docker-compose down

# With Podman
podman compose down
```

## ⚠️ Python Support in the Code Node

The default n8n Docker image is based on **Alpine Linux**, a lightweight Linux distribution designed to be as minimal as possible. Because of this, **Python is not included** in the image — Alpine only ships with the bare essentials needed to run n8n (Node.js). The Code node runs **JavaScript (Node.js)** natively, which is sufficient for most automation workflows.

If you need Python support in the Code node, you must build a **custom Docker image**:

```dockerfile
FROM docker.n8n.io/n8nio/n8n

USER root
RUN apk add --no-cache python3 py3-pip
USER node
```

Then update `docker-compose.yaml` to build from the Dockerfile instead of pulling the image:

```yaml
n8n:
    container_name: n8n
    build: .
    # image: docker.n8n.io/n8nio/n8n  # comment this out
```

> [!NOTE]
> JavaScript is the most widely used language in n8n's Code node and is recommended for most use cases.
