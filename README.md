# 🚀 Local DevOps Stack

A comprehensive, self-hosted DevOps stack running locally with Docker Compose. Designed to be accessed securely via WireGuard from a remote proxy.

## 📚 Practice Resources

- **Kubernetes + ArgoCD**: [https://github.com/root404onion/kubernates.git](https://github.com/root404onion/kubernates.git) - A repository for practicing **ArgoCD** GitOps workflows.

## 🏗️ Architecture

This stack is designed to run on a local machine (e.g., a home server) while being accessible publicly or privately via a secure WireGuard tunnel.

```mermaid
graph TD
    User([User / Internet]) -->|HTTPS| DO["Digital Ocean VM<br/>(Nginx Proxy)"]
    
    subgraph "Secure Tunnel"
        DO <==>|WireGuard VPN<br/>10.0.0.x| LocalMachine[Local Server]
    end
    
    subgraph "Local Docker Stack"
        LocalMachine -->|Port 3000| Gitea
        LocalMachine -->|Port 9001| MinIO["MinIO Console"]
        LocalMachine -->|Port 8080| Jenkins
        LocalMachine -->|Port 8081| Nexus
        LocalMachine -->|Port 8200| Vault
        
        Gitea -.-> Postgres
        Gitea -.-> Redis
        Gitea -.-> MinIO
    end
```

## 📦 Services

| Service | Local Port | Description |
|---------|------------|-------------|
| **Gitea** | `3000` | SCM, Git hosting |
| **Gitea SSH** | `2222` | SSH Passthrough for Git |
| **MinIO** | `9000` (API)<br/>`9001` (UI) | S3-compatible object storage |
| **Jenkins** | `8080` | CI/CD Automation |
| **Nexus** | `8081` | Artifact Registry (Maven, Docker, etc.) |
| **Vault** | `8200` | Secrets Management |

## 🛠️ Prerequisites

- **Docker Desktop** or Docker Engine + Compose
- **WireGuard**: Required for remote access.

## 🔒 WireGuard Setup (Remote Access)

To connect your local machine to the remote infrastructure:

1.  **Install WireGuard**:
    ```bash
    curl -fsSL https://code.xeze.org/dev/wireguard/raw/branch/main/install.sh | sudo bash
    ```

2.  **Add User Clients**:
    ```bash
    wget https://code.xeze.org/dev/wireguard/raw/branch/main/add.sh
    chmod +x add.sh
    sudo ./add.sh
    ```

3.  **Get Client Config**:
    ```bash
    cat /etc/wireguard/clients/client1.conf
    ```

4.  **Generate QR Code for Mobile**:
    ```bash
    qrencode -t ansiutf8 < /etc/wireguard/clients/client1.conf
    ```

## 🚀 Setup Guide

1.  **Clone the repository**:
    ```bash
    git clone https://github.com/root404onion/git-ops.git
    cd git-ops
    ```

2.  **Configure Environment**:
    copy the example file and update your secrets.
    ```bash
    cp .env.example .env
    ```
    > **Note**: Update `GITEA_DOMAIN`, `POSTGRES_PASSWORD`, etc., in `.env`.

3.  **Start the Stack**:
    ```bash
    docker-compose up -d
    ```

4.  **Access Services**:
    Open your browser to `http://localhost:3000` to set up Gitea.

## 📂 Data Storage

All data is persisted locally in the `./data` directory (ignored by git):
- `./data/gitea`: Repositories and user data
- `./data/postgres`: Database files
- `./data/minio`: S3 Objects (LFS, Packages)

## 🔒 Security

- **No Local SSL**: This stack exposes **HTTP** ports. SSL termination should be handled by your reverse proxy (e.g., Nginx on your VPS).
- **Secrets**: Provide secrets via the `.env` file. Do not commit `.env` to version control.
