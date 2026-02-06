# 🧠 Gitea + PostgreSQL + MinIO Architecture - Visual Guide

## 🏗️ The Big Picture - What Lives Where?

```mermaid
flowchart TB
    subgraph YOUR_COMPUTER["🖥️ YOUR COMPUTER"]
        subgraph DOCKER["🐳 Docker Compose Stack"]
            GITEA["🦊 Gitea<br/>Web UI + Git Server<br/>Port 3000 & 2222"]
            POSTGRES[("🐘 PostgreSQL<br/>Database<br/>Port 5432")]
            MINIO["📦 MinIO<br/>Object Storage<br/>Port 9000 & 9001"]
            RUNNER["🏃 Runner<br/>CI/CD Jobs"]
        end
        
        subgraph FOLDERS["📁 Your Disk (e:\\gitealocal\\data\\)"]
            F_GITEA["📂 gitea/<br/>Git repos + config"]
            F_POSTGRES["📂 postgres/<br/>Database files"]
            F_MINIO["📂 minio/<br/>LFS + Attachments"]
            F_RUNNER["📂 runner/<br/>Runner config"]
        end
    end
    
    GITEA --> F_GITEA
    POSTGRES --> F_POSTGRES
    MINIO --> F_MINIO
    RUNNER --> F_RUNNER
```

---

## 🎬 Scenario 1: You Push Code to Gitea

**Imagine:** You run `git push origin main` to upload your project

```mermaid
flowchart LR
    subgraph YOU["👨‍💻 You"]
        GIT["git push origin main"]
    end
    
    subgraph GITEA_SERVER["🦊 Gitea Container"]
        RECEIVE["Receive Push"]
        SAVE_REPO["Save to /data"]
        UPDATE_DB["Update Database"]
    end
    
    subgraph STORAGE["💾 Where It Goes"]
        DISK["📂 ./data/gitea/<br/>Your actual .git files"]
        PG[("🐘 PostgreSQL<br/>Repo metadata:<br/>- Repo name<br/>- Owner<br/>- Stars<br/>- Issues")]
    end
    
    GIT -->|"SSH port 2222"| RECEIVE
    RECEIVE --> SAVE_REPO
    SAVE_REPO -->|"Commits, branches, files"| DISK
    RECEIVE --> UPDATE_DB
    UPDATE_DB -->|"Repo info, permissions"| PG
```

### 📍 Where does your code actually go?

| What | Where | Example |
|------|-------|---------|
| Your actual code files | `./data/gitea/git/repositories/` | `./data/gitea/git/repositories/yourname/project.git/` |
| Repo metadata (name, stars, issues) | PostgreSQL | Stored in database tables |

---

## 🎬 Scenario 2: You Upload a Large File (LFS)

**Imagine:** You're storing a 500MB video file with Git LFS

```mermaid
flowchart TD
    subgraph YOU["👨‍💻 You"]
        PUSH["git lfs push<br/>big_video.mp4 (500MB)"]
    end
    
    subgraph GITEA_SERVER["🦊 Gitea"]
        LFS_HANDLER["LFS Handler"]
        DECISION{"File type?"}
    end
    
    subgraph STORAGE["💾 Storage"]
        POINTER["📂 ./data/gitea/<br/>Tiny pointer file<br/>(just 130 bytes)"]
        BLOB["📦 MinIO<br/>./data/minio/<br/>Actual 500MB video"]
    end
    
    PUSH --> LFS_HANDLER
    LFS_HANDLER --> DECISION
    DECISION -->|"Pointer file"| POINTER
    DECISION -->|"Big binary blob"| BLOB
    
    style BLOB fill:#f9a825
    style POINTER fill:#4caf50
```

### 🤔 Why use MinIO for LFS?

| Without MinIO | With MinIO |
|---------------|------------|
| 500MB video stored in git folder | 500MB video in MinIO bucket |
| Git operations become SLOW 🐌 | Git stays FAST 🚀 |
| Backup is complicated | Easy to backup separately |

---

## 🎬 Scenario 3: You Attach an Image to an Issue

**Imagine:** You drag & drop a screenshot into a GitHub Issue

```mermaid
flowchart LR
    subgraph YOU["👨‍💻 You"]
        DRAG["🖼️ Drag screenshot.png<br/>into Issue comment"]
    end
    
    subgraph GITEA_SERVER["🦊 Gitea"]
        UPLOAD["Upload Handler"]
    end
    
    subgraph MINIO_BUCKET["📦 MinIO (./data/minio/)"]
        ATTACHMENT["gitea-bucket/<br/>attachments/<br/>abc123-screenshot.png"]
    end
    
    subgraph POSTGRES["🐘 PostgreSQL"]
        ISSUE_DB["Issue #42<br/>body: '...see image...'<br/>attachment_id: abc123"]
    end
    
    DRAG --> UPLOAD
    UPLOAD -->|"Store binary"| ATTACHMENT
    UPLOAD -->|"Save reference"| ISSUE_DB
    
    style ATTACHMENT fill:#f9a825
```

---

## 🎬 Scenario 4: CI/CD Pipeline Runs

**Imagine:** You push code and it triggers a build

```mermaid
flowchart TD
    subgraph YOU["👨‍💻 You"]
        PUSH2["git push"]
    end
    
    subgraph DOCKER_NETWORK["🌐 Internal Docker Network (gitea)"]
        GITEA2["🦊 Gitea<br/>http://gitea:3000"]
        RUNNER2["🏃 Runner"]
        
        GITEA2 <-->|"Super fast!<br/>No internet hop"| RUNNER2
    end
    
    subgraph OUTSIDE["🌍 Outside World"]
        BROWSER["Your Browser<br/>http://localhost:3000"]
    end
    
    PUSH2 --> GITEA2
    GITEA2 -->|"1. Trigger workflow"| RUNNER2
    RUNNER2 -->|"2. Clone repo"| GITEA2
    RUNNER2 -->|"3. Run tests"| RUNNER2
    RUNNER2 -->|"4. Report status"| GITEA2
    
    BROWSER -.->|"View results"| GITEA2
    
    style DOCKER_NETWORK fill:#e3f2fd
```

### ⚡ Why is the Runner on the same network?

```
┌─────────────────────────────────────────────────────────────┐
│  WITHOUT same network:                                       │
│                                                              │
│  Runner ──→ Internet ──→ localhost:3000 ──→ Gitea           │
│         🐌 SLOW (100ms+ latency)                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  WITH same network (your setup):                             │
│                                                              │
│  Runner ──→ gitea:3000 ──→ Gitea                            │
│         ⚡ INSTANT (<1ms latency)                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Complete Data Flow Summary

```mermaid
flowchart TB
    subgraph ACTIONS["👨‍💻 Your Actions"]
        A1["git push code"]
        A2["git lfs push bigfile"]
        A3["Attach image to issue"]
        A4["Create issue/PR"]
        A5["Trigger CI pipeline"]
    end
    
    subgraph CONTAINERS["🐳 Containers"]
        GITEA3["🦊 Gitea"]
        PG3[("🐘 PostgreSQL")]
        MINIO3["📦 MinIO"]
        RUNNER3["🏃 Runner"]
    end
    
    subgraph DISK["📁 Your Disk"]
        D1["./data/gitea/"]
        D2["./data/postgres/"]
        D3["./data/minio/"]
        D4["./data/runner/"]
    end
    
    A1 --> GITEA3 --> D1
    A2 --> GITEA3 --> MINIO3 --> D3
    A3 --> GITEA3 --> MINIO3 --> D3
    A4 --> GITEA3 --> PG3 --> D2
    A5 --> GITEA3 --> RUNNER3 --> D4
    
    style D1 fill:#4caf50
    style D2 fill:#2196f3
    style D3 fill:#f9a825
    style D4 fill:#9c27b0
```

---

## 📊 What Stores What - Simple Table

| Container | What It Stores | Disk Location | Example Data |
|-----------|---------------|---------------|--------------|
| 🦊 **Gitea** | Git repositories, config | `./data/gitea/` | Your actual code, branches, commits |
| 🐘 **PostgreSQL** | Metadata & relationships | `./data/postgres/` | Users, repos, issues, PRs, comments, permissions |
| 📦 **MinIO** | Large binary files | `./data/minio/` | LFS files, issue attachments, avatars |
| 🏃 **Runner** | CI/CD config & cache | `./data/runner/` | Runner registration, job cache |

---

## 🎯 Real-World Analogy

Think of it like a **Library** 📚:

```
┌────────────────────────────────────────────────────────────────┐
│  🏛️ LIBRARY (Gitea)                                           │
│                                                                │
│  📚 Bookshelves = ./data/gitea/                                │
│     (Your actual books/code)                                   │
│                                                                │
│  🗃️ Card Catalog = PostgreSQL                                  │
│     (Where is each book? Who borrowed it? Reviews?)            │
│                                                                │
│  📦 Storage Warehouse = MinIO                                  │
│     (Giant maps, posters, videos - too big for shelves)        │
│                                                                │
│  🏃 Delivery Truck = Runner                                    │
│     (Picks up code, runs tests, delivers results)              │
└────────────────────────────────────────────────────────────────┘
```

---

## ❓ Quick FAQ

**Q: If I delete the database, do I lose my code?**  
A: NO! Your code is in `./data/gitea/`. But you'll lose issues, PRs, users, permissions.

**Q: If I delete MinIO, do I lose my code?**  
A: NO! Your code is in `./data/gitea/`. But you'll lose LFS files and attachments.

**Q: Can I backup just the code?**  
A: YES! Just backup `./data/gitea/git/repositories/`

**Q: Why not store everything in one place?**  
A: Performance! Each tool is optimized for its job:
- Git = fast code versioning
- PostgreSQL = fast queries on structured data
- MinIO = efficient large file storage
