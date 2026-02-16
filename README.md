# n8n with Ngrok Tunnel

Read this README in [Português 🇧🇷](README.pt-BR.md)

This repository contains a Docker Compose setup for running **n8n** with **Ngrok** as a tunneling service. n8n is a workflow automation tool that allows you to connect different services and APIs. Ngrok exposes local servers behind NATs and firewalls to the public internet over secure tunnels.

> Based on [joffcom/n8n-ngrok](https://github.com/joffcom/n8n-ngrok) with additional configuration for HTTPS and OAuth support.

## Prerequisites

Before you begin, ensure you have the following installed:
- [Docker](https://docs.docker.com/get-docker/) or [Podman](https://podman.io/getting-started/installation)
- [Docker Compose](https://docs.docker.com/compose/install/) (even if using Podman)

## Setup

### 1. Clone the Repository

```bash
git clone https://github.com/nelsonsoares/n8n-ngrok-https.git
cd n8n-ngrok-https
```

### 2. Ngrok Authentication

You need an Ngrok account. If you don't have one, create it at [ngrok.com](https://ngrok.com/). After creating an account, get your auth token from the Ngrok dashboard.

### 3. Reserve a Permanent Domain

By default, Ngrok may provide an ephemeral domain. For this setup to work reliably, it is highly recommended to **reserve a permanent domain** in your Ngrok Dashboard under **Cloud Edge > Domains**. 

If you already have a reserved domain, copy it; otherwise, Ngrok will assign one to you upon first connection, but you should eventually reserve it to avoid changing your `.env` every time you restart.

### 4. Configure Environment Variables

1. Copy the template file to create your own `.env` file:
   ```bash
   cp .env-template .env
   ```
2. Open the `.env` file and fill in your details:

```sh
TIMEZONE=America/Sao_Paulo
NGROK_TOKEN=your_ngrok_auth_token_here
URL=https://your-domain.ngrok-free.dev
NGROK_DOMAIN=your-domain.ngrok-free.dev

# Database (PostgreSQL with pgvector)
POSTGRES_USER=n8n
POSTGRES_PASSWORD=n8n
POSTGRES_DB=n8n

# Redis (for Queue Mode)
REDIS_PASSWORD=n8n
```

> [!CAUTION]
> **SECURITY WARNING:** Never commit your `.env` file to version control. This project includes a `.gitignore` file that excludes `.env` by default. If you accidently leak your `NGROK_TOKEN` or `NGROK_DOMAIN`, rotate your secrets immediately in the Ngrok dashboard.

> [!IMPORTANT]
> - The `URL` variable **must** include the `https://` prefix for OAuth to work.
> - The `NGROK_DOMAIN` variable is now passed **dynamically** to the Ngrok container via `docker-compose.yaml`. This eliminates the need for an external `ngrok.yml` file, improving compatibility with Podman (avoiding rootless volume permission issues).

## Key Configuration Details

### Professional Scaling & AI Support

This setup is configured for production-like studies, featuring:

- **PostgreSQL (with pgvector)**: Replaces SQLite for robust data handling. The `pgvector` extension is **automatically enabled** on startup via `init-db.sql`, allowing you to use Postgres as a Vector Database for AI/LLM workflows (embeddings).
- **Redis & Queue Mode**: Separates the n8n main instance from executions. The `n8n-worker` service handles the actual workload, ensuring the editor remains responsive even under heavy load.
- **Management Tools (Browser-based)**: 
    - **pgAdmin**: Manage Postgres via browser at [http://localhost:5050](http://localhost:5050).
    - **Redis Commander**: Manage Redis via browser at [http://localhost:8081](http://localhost:8081).

### 💡 Understanding Access & Security

For beginners, it's important to understand there are two different "layers" of access:

1.  **Management UI (The Software)**: When you open pgAdmin in the browser, you are logging into the *management tool*. Use the default `admin@admin.com` / `admin` credentials.
2.  **Database Services (The Data)**: Once inside the management tool (or when configuring a node in n8n), you connect to the *actual database* using the credentials defined in your `.env` (e.g., `POSTGRES_USER=n8n`).

> [!NOTE]
> **Connections from Windows**: If using a desktop tool (like DBeaver), use `localhost` and Port `5433`.
> **Connections within n8n**: If configuring a Postgres node in n8n, use Host `postgres` and Port `5432`.

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
docker-compose up -d
```

## Accessing n8n

After starting the services, access n8n by navigating to your Ngrok domain in your browser:

```
https://your-domain.ngrok-free.dev
```

## Stopping the Application

```bash
docker-compose down
```

> [!TIP]
> **STABILITY TIP:** If you encounter connection issues or configuration errors (like `ERR_NGROK_9034`), always use `docker-compose down` to fully stop and remove containers and virtual networks before starting them again. This ensures a clean state for the dynamic configuration.


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
