
# VORTEX — THE COMPLETE REFERENCE
### Cloud-Native Static Website Deployment Platform

---

> **What is this document?**
> This is a 100+ page comprehensive reference for every technology, architecture decision, debugging story, and interview-ready explanation used in the Vortex project. Every section answers WHY, HOW, WHAT HAPPENS, and includes tradeoffs, interview questions, and counter questions.

---

# TABLE OF CONTENTS

| Section | Title | Page |
|---------|-------|------|
| 1 | [Project Overview](#section-1-project-overview) | — |
| 2 | [Business Problem](#section-2-business-problem) | — |
| 3 | [Architecture](#section-3-architecture) | — |
| 4 | [Request Flow](#section-4-request-flow) | — |
| 5 | [Deployment Flow](#section-5-deployment-flow) | — |
| 6 | [Infrastructure](#section-6-infrastructure) | — |
| 7 | [Terraform](#section-7-terraform) | — |
| 8 | [AWS](#section-8-aws) | — |
| 9 | [Docker](#section-9-docker) | — |
| 10 | [Docker Compose](#section-10-docker-compose) | — |
| 11 | [Nginx](#section-11-nginx) | — |
| 12 | [Backend](#section-12-backend) | — |
| 13 | [Frontend](#section-13-frontend) | — |
| 14 | [MongoDB](#section-14-mongodb) | — |
| 15 | [Kafka](#section-15-kafka) | — |
| 16 | [ClickHouse](#section-16-clickhouse) | — |
| 17 | [CI/CD](#section-17-cicd) | — |
| 18 | [GitHub Actions](#section-18-github-actions) | — |
| 19 | [Ansible](#section-19-ansible) | — |
| 20 | [Security](#section-20-security) | — |
| 21 | [Failure Scenarios](#section-21-failure-scenarios) | — |
| 22 | [Scalability](#section-22-scalability) | — |
| 23 | [Tradeoffs](#section-23-tradeoffs) | — |
| 24 | [Why This Technology](#section-24-why-this-technology) | — |
| 25 | [Interview Questions](#section-25-interview-questions) | — |
| 26 | [Counter Questions](#section-26-counter-questions) | — |
| 27 | [System Design](#section-27-system-design) | — |
| 28 | [Production Improvements](#section-28-production-improvements) | — |
| 29 | [Debugging Stories](#section-29-debugging-stories) | — |
| 30 | [Real Problems Faced](#section-30-real-problems-faced) | — |

---
---

# SECTION 1: PROJECT OVERVIEW

## What is Vortex?

Vortex is a **cloud-native deployment service** inspired by **Vercel**, designed for **static website hosting**. It is a full-stack platform that allows developers to:

1. **Register and authenticate** with the platform.
2. **Connect their GitHub profile** to fetch repositories.
3. **Select a repository and branch** to deploy.
4. **Trigger a containerized build** (the user's repo is cloned, dependencies installed, and a production build is created — all inside an ephemeral Docker container).
5. **Stream real-time build logs** through a Kafka pipeline to a ClickHouse columnar database.
6. **Upload the build output to AWS S3** for static hosting.
7. **View deployment history and live URLs** from a modern React dashboard.

## Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Frontend** | React 19 + Vite + TailwindCSS 4 | SPA with modern UI, fast HMR |
| **State Management** | Redux Toolkit + Redux Persist | Global auth state, session persistence |
| **Backend** | Express 5 (Node.js 18/20) | REST API, authentication, deployment orchestration |
| **Database** | MongoDB 6.0 (Mongoose ODM) | User data, deployment metadata |
| **Message Queue** | Apache Kafka 3.8 (3-node KRaft cluster) | Real-time build log streaming |
| **Analytics DB** | ClickHouse 23.4 | Columnar storage for build logs with sub-second queries |
| **Object Storage** | AWS S3 | Static site artifact hosting |
| **Reverse Proxy** | Nginx 1.25 Alpine | API gateway, gzip, security headers |
| **Containerization** | Docker + Docker Compose | Service orchestration, isolated builds |
| **Infrastructure** | Terraform (AWS Provider ~5.0) | Declarative EC2/VPC/IAM provisioning |
| **Configuration** | Ansible (Roles-based) | Automated server setup and deployment |
| **CI** | GitHub Actions (CI Pipeline) | Lint, test, build, security scanning |
| **CD** | GitHub Actions + Ansible | Automated production deployment |
| **Monitoring** | Redpanda Console | Kafka topic inspection and monitoring UI |
| **Security Scanning** | Trivy, Gitleaks, Snyk | Filesystem, secret, and dependency vulnerability scanning |

## Repository Structure

```
Vortex/
├── backend/                    # Express.js REST API
│   ├── controllers/            # Route handlers
│   │   ├── auth.controller.js
│   │   ├── deploy.controller.js
│   │   ├── git.controller.js
│   │   ├── log.controller.js
│   │   └── user.controller.js
│   ├── models/                 # Mongoose schemas
│   │   ├── deployment.model.js
│   │   └── user.model.js
│   ├── middlewares/            # GitHub integration middleware
│   ├── routes/                 # Express route definitions
│   └── server.js               # Application entry point
├── frontend/                   # React + Vite SPA
│   └── src/
│       ├── components/         # Header, Footer, PrivateRoute, AutoLogout
│       ├── pages/              # Auth, Home, Dashboard, Deploy, Settings
│       ├── redux/              # Redux Toolkit store + UserSlice
│       └── services/           # Axios API client
├── build-server/               # Ephemeral build container
│   ├── Dockerfile              # Ubuntu + Node + Java + Git
│   ├── main.sh                 # Entrypoint: clone, checkout, env, build
│   └── script.js               # Build orchestration + S3 upload + Kafka logging
├── services/                   # Docker Compose stack
│   ├── docker-compose.yml      # 8 services orchestrated
│   ├── init-table.sql          # ClickHouse schema + Kafka consumer + materialized view
│   └── create-topic.sh         # Kafka topic initialization
├── nginx/                      # Reverse proxy
│   ├── Dockerfile
│   └── nginx.conf
├── infra/
│   ├── terraform/              # AWS infrastructure as code
│   │   ├── ec2.tf, iam.tf, networking.tf, security.tf
│   │   ├── variables.tf, outputs.tf, versions.tf
│   │   └── user_data.sh        # EC2 bootstrap script
│   └── ansible/                # Configuration management
│       ├── playbook.yml
│       ├── inventory.ini
│       └── roles/vortex/       # Tasks, handlers, templates, defaults
├── scripts/                    # Operational scripts
│   ├── deploy.sh, rollback.sh
│   ├── health-check.sh, backup.sh
├── .github/workflows/
│   ├── ci.yml                  # Continuous integration
│   └── cd.yml                  # Continuous deployment
├── Dockerfile                  # Application container (frontend + backend)
└── package.json                # Root workspace
```

---
---

# SECTION 2: BUSINESS PROBLEM

## The Problem

Modern developers need a fast, reliable way to deploy static websites from their GitHub repositories. Existing solutions like Vercel, Netlify, and GitHub Pages have limitations:

| Pain Point | Description |
|-----------|-------------|
| **Vendor Lock-in** | Vercel and Netlify lock you into their proprietary build systems. You don't understand what happens underneath. |
| **Opaque Build Process** | Users can't see exactly what happens during the build. Logs are often delayed or incomplete. |
| **Limited Customization** | Framework detection is rigid. Custom build pipelines require workarounds. |
| **Cost at Scale** | Free tiers are limited. Bandwidth and build minutes get expensive fast. |
| **No Learning** | Using managed services means you never learn about Docker, Kafka, AWS, Terraform, CI/CD, Nginx, or any real infrastructure. |

## The Solution: Vortex

Vortex solves this by being a **self-hosted, fully transparent deployment platform** where:

1. **Every build happens in an isolated Docker container** — no shared build environments, no side effects.
2. **Build logs stream in real-time** through Kafka to ClickHouse, giving sub-second query performance on log data.
3. **The entire infrastructure is code** — Terraform provisions, Ansible configures, GitHub Actions deploys.
4. **Users own their data** — MongoDB stores user accounts, S3 stores artifacts, ClickHouse stores logs.
5. **Everything is observable** — Redpanda Console for Kafka, ClickHouse for logs, health endpoints for monitoring.

## Who Uses It?

- Developers who want a Vercel-like experience on their own infrastructure.
- Engineering teams learning deployment pipelines, containerization, and cloud architecture.
- Anyone deploying static React/Vite/Next.js/plain HTML websites from GitHub.

---
---

# SECTION 3: ARCHITECTURE

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          AWS EC2 Instance                           │
│                        (t3.large, Ubuntu 24.04)                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                     Docker Compose Stack                       │ │
│  │                                                                │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                    │ │
│  │  │ Kafka-1  │  │ Kafka-2  │  │ Kafka-3  │  (KRaft Cluster)   │ │
│  │  │ Broker+  │  │ Broker+  │  │ Broker+  │                    │ │
│  │  │ Ctrl     │  │ Ctrl     │  │ Ctrl     │                    │ │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘                    │ │
│  │       │              │              │                          │ │
│  │       └──────────────┼──────────────┘                          │ │
│  │                      │                                         │ │
│  │                      ▼                                         │ │
│  │              ┌──────────────┐                                  │ │
│  │              │  ClickHouse  │  (Kafka Consumer → MergeTree)    │ │
│  │              └──────────────┘                                  │ │
│  │                                                                │ │
│  │  ┌──────────────┐     ┌──────────────┐                        │ │
│  │  │   MongoDB    │     │   Redpanda   │                        │ │
│  │  │   (Users,    │     │   Console    │                        │ │
│  │  │  Deploys)    │     │  (Kafka UI)  │                        │ │
│  │  └──────┬───────┘     └──────────────┘                        │ │
│  │         │                                                      │ │
│  │         ▼                                                      │ │
│  │  ┌──────────────────────────────────┐                          │ │
│  │  │       Vortex Application         │                          │ │
│  │  │  ┌────────────┐ ┌─────────────┐  │                         │ │
│  │  │  │  Frontend   │ │   Backend   │  │                         │ │
│  │  │  │  React+Vite │ │  Express.js │  │                         │ │
│  │  │  │  :3000      │ │  :5000      │  │                         │ │
│  │  │  └────────────┘ └──────┬──────┘  │                         │ │
│  │  └────────────────────────┼─────────┘                          │ │
│  │                           │                                    │ │
│  │                    docker run -d                                │ │
│  │                           │                                    │ │
│  │                           ▼                                    │ │
│  │              ┌──────────────────────┐                          │ │
│  │              │  Build Server        │  (Ephemeral Container)   │ │
│  │              │  • Clone Repo        │                          │ │
│  │              │  • npm install       │                          │ │
│  │              │  • npm run build     │                          │ │
│  │              │  • Upload to S3      │                          │ │
│  │              │  • Log to Kafka      │                          │ │
│  │              └──────────────────────┘                          │ │
│  │                                                                │ │
│  │  ┌──────────────────┐                                          │ │
│  │  │     Nginx        │  (Reverse Proxy, :80)                    │ │
│  │  │  /api → :5000    │                                          │ │
│  │  │  /health → :5000 │                                          │ │
│  │  │  /console → :8080│                                          │ │
│  │  └──────────────────┘                                          │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  VPC: 10.0.0.0/16  │  Subnet: 10.0.1.0/24  │  Elastic IP          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                       ┌──────────────┐
                       │    AWS S3     │
                       │  (Static     │
                       │   Assets)    │
                       └──────────────┘
```

## Microservices Breakdown

The Docker Compose stack runs **8 services** on a single EC2 instance:

| # | Service | Container Name | Port(s) | Role |
|---|---------|---------------|---------|------|
| 1 | Kafka Broker 1 | `kafka-1` | 19092 | Message broker (KRaft mode, combined broker+controller) |
| 2 | Kafka Broker 2 | `kafka-2` | 29092 | Replica broker for fault tolerance |
| 3 | Kafka Broker 3 | `kafka-3` | 39092 | Replica broker for quorum voting |
| 4 | Topic Initializer | `kafka-topic-init` | — | One-shot container to create `build-logs` topic |
| 5 | Redpanda Console | `kafka-console` | 8080 | Web UI for Kafka monitoring |
| 6 | ClickHouse | `clickhouse` | 8123, 9000 | Columnar analytics DB (log storage) |
| 7 | MongoDB | `mongodb` | 27017 | Document DB (users, deployments) |
| 8 | Vortex App | `vortex-app` | 3005→3000, 5005→5000 | Frontend + Backend |
| 9 | Nginx | `vortex-nginx` | 80 | Reverse proxy gateway |

**Plus dynamically spawned:**

| Service | Container Name | Lifecycle |
|---------|---------------|-----------|
| Build Server | `{deployment-id}` | Ephemeral — created per deployment, destroyed after completion |

## Why This Architecture?

**Separation of Concerns**: Each service does one thing well. MongoDB handles transactional data, ClickHouse handles analytics, Kafka handles real-time streaming, S3 handles static file storage.

**Isolation**: Each build runs in its own Docker container. A failing build cannot crash the main application or affect other users' builds.

**Observability**: Every build log message goes through Kafka and lands in ClickHouse. You can query any deployment's logs with sub-second latency, even months later.

**Reproducibility**: The entire infrastructure is defined as code (Terraform + Ansible + Docker Compose). You can destroy everything and recreate it from scratch in under 10 minutes.

---
---

# SECTION 4: REQUEST FLOW

## User Registration Flow

```
User (Browser)
    │
    ├── POST /api/auth/register
    │   {username, fullname, email, password, githubProfile}
    │
    ▼
Nginx (:80)
    │
    ├── proxy_pass → http://backend_nodes (application:5000)
    │
    ▼
Express Backend (:5000)
    │
    ├── Rate Limiter (5000 req/15min)
    ├── Helmet (Security Headers)
    ├── CORS Check
    │
    ▼
auth.controller.js → registerUser()
    │
    ├── Validate: All fields present?
    ├── Check: Email already exists? (User.findOne)
    ├── Check: Username already exists? (User.findOne)
    ├── Hash password: bcrypt.hash(password, 10)
    ├── Clean GitHub profile: strip "https://github.com/"
    ├── Save: new User({...}).save()
    │
    ▼
MongoDB (users collection)
    │
    └── Document created with timestamps
```

**Why bcrypt with salt rounds 10?** Salt rounds of 10 means 2^10 = 1024 iterations of the hashing algorithm. This makes brute-force attacks computationally expensive (each hash takes ~100ms) while keeping registration fast enough for users. Higher values like 12 or 14 would be more secure but slower.

**Why strip the GitHub URL prefix?** Users might enter their GitHub profile as `https://github.com/eshwarsai` or just `eshwarsai`. The controller normalizes this to just the username so the Octokit API can use it directly.

## User Login Flow

```
User (Browser)
    │
    ├── POST /api/auth/login
    │   {email, password}
    │
    ▼
auth.controller.js → loginUser()
    │
    ├── Find user by email (User.findOne)
    ├── Compare: bcrypt.compare(password, user.password)
    ├── Generate JWT: jwt.sign({id: user._id}, JWT_SECRET, {expiresIn: '7d'})
    ├── Strip password from response
    │
    ▼
Response → {token, user}
    │
    ▼
Frontend (Redux)
    │
    ├── Dispatch loginSuccess({user, token})
    ├── Redux Persist saves to localStorage
    └── AutoLogout component starts 7-day timer
```

**Why JWT with 7-day expiry?** A 7-day expiry balances security with user experience. Users don't need to login every day, but stale tokens expire within a week. In production, you'd add refresh tokens for seamless re-authentication.

## Deployment Trigger Flow

```
User clicks "Deploy" on Dashboard
    │
    ├── POST /api/deploy/start
    │   {repo, branch, username, deploymentId, envVars}
    │
    ▼
deploy.controller.js → deployProject()
    │
    ├── 1. Validate required fields
    │
    ├── 2. Check if build-server Docker image exists
    │      docker image inspect vortex-build-server:latest
    │      └── If missing → docker build -t vortex-build-server:latest /home/ubuntu/vortex/build-server
    │
    ├── 3. Remove any existing container with same deployment ID
    │      docker rm -f {deploymentId}
    │
    ├── 4. docker run -d
    │      --name {deploymentId}
    │      --network deployment
    │      -e REPO={repo}
    │      -e BRANCH={branch}
    │      -e DEPLOYMENT_ID={deploymentId}
    │      -e KAFKA_BROKER=kafka-1:19092
    │      -e ACCESS_KEY_ID={aws_key}
    │      -e SECRET_ACCESS_KEY={aws_secret}
    │      -e S3_BUCKET={bucket}
    │      vortex-build-server:latest
    │
    ▼
Response → {message: "Deployment started", deploymentId}
```

**Why `--network deployment`?** The build server container needs to communicate with Kafka brokers to send logs. All services run on a custom Docker bridge network called `deployment`. By joining this network, the build container can resolve `kafka-1:19092` via Docker's internal DNS.

**Why ephemeral containers?** Each deployment runs in its own isolated container. If a user's project has a bad `npm install` that corrupts `node_modules` or runs malicious code, it's confined to that container. When the build finishes (success or failure), the container exits. This is the same pattern used by Vercel, GitHub Actions, and all major CI/CD platforms.

## Build Log Flow (Real-Time Pipeline)

```
Build Server Container
    │
    ├── publishLog("Installing dependencies...", "INFO")
    │
    ▼
Kafka Producer (kafkajs)
    │
    ├── Send to topic: "build-logs"
    │   Message: {
    │     deployment_id: "abc123",
    │     log_message: "Installing dependencies...",
    │     log_level: "INFO"
    │   }
    │
    ▼
Kafka Cluster (3 brokers)
    │
    ├── kafka-1:19092 (leader)
    ├── kafka-2:19092 (replica)
    ├── kafka-3:19092 (replica)
    │
    ▼
ClickHouse Kafka Engine Table (logs.log_queue)
    │
    ├── Consumes from topic "build-logs"
    ├── Consumer group: "clickhouse-consumer"
    ├── Format: JSONEachRow
    │
    ▼
Materialized View (logs.kafka_queue)
    │
    ├── Transforms: adds timestamp (Asia/Kolkata), generates UUID
    ├── Writes to: logs.build_logs (MergeTree table)
    │
    ▼
Frontend polls: GET /api/logs/{deploymentId}
    │
    ▼
log.controller.js → ClickHouse query
    │
    ├── SELECT log_uuid, deployment_id, log_message, log_level, created_at
    │   FROM build_logs
    │   WHERE deployment_id = '{id}'
    │   ORDER BY created_at ASC
    │
    ▼
Logs displayed in real-time on Deploy page
```

**Why this pipeline instead of just writing logs to MongoDB?** Three reasons:

1. **Write throughput**: Kafka can handle millions of messages per second. MongoDB would become a bottleneck under high log volume.
2. **Decoupling**: The build server doesn't need to know about ClickHouse. It just sends to Kafka. ClickHouse consumes independently.
3. **Analytics**: ClickHouse is a columnar database optimized for `WHERE` + `ORDER BY` queries on time-series data. Querying 100,000 log lines takes milliseconds, not seconds.

---
---

# SECTION 5: DEPLOYMENT FLOW

## Complete End-to-End Deployment Flow

Here is every single step that happens when a user deploys a website through Vortex:

### Step 1: User → Frontend

**What happens**: User logs in, navigates to Home page, selects a GitHub repository, picks a branch, optionally adds environment variables, and clicks "Deploy".

**How**: The React frontend calls `POST /api/deploy/start` with the deployment payload.

**Why**: The frontend acts as the user interface layer. It doesn't do any building or deploying — it only sends instructions to the backend.

### Step 2: Frontend → Backend (API Call)

**What happens**: The HTTP request travels through Nginx reverse proxy to the Express backend.

**How**: Nginx receives on port 80, sees `/api/` prefix, proxy_passes to `application:5000` (the backend container).

**Why**: Nginx acts as the edge gateway. It handles compression, security headers, and routing. The Express backend never exposes itself directly to the internet.

### Step 3: Backend → Authentication Check

**What happens**: The backend validates the request has required fields (repo, branch, username, deploymentId).

**How**: Field validation in `deployProject()` controller.

**Why**: Without validation, incomplete requests could spawn broken containers that waste resources.

### Step 4: Backend → Docker Image Check

**What happens**: The backend checks if `vortex-build-server:latest` Docker image exists on the host.

**How**: `docker image inspect vortex-build-server:latest` — if this fails, it runs `docker build -t vortex-build-server:latest /home/ubuntu/vortex/build-server` to build it on-the-fly.

**Why**: The build server image might not exist after a fresh deployment or if someone pruned Docker images. Auto-building ensures the system is self-healing.

### Step 5: Backend → Docker Container Launch

**What happens**: A new Docker container is spawned from the `vortex-build-server:latest` image with all environment variables injected.

**How**: `docker run -d --name {deploymentId} --network deployment -e REPO=... -e KAFKA_BROKER=kafka-1:19092 -e S3_BUCKET=... vortex-build-server:latest`

**What happens inside the container**:

### Step 6: Build Server → Clone Repository

**What happens**: The `main.sh` entrypoint clones the user's GitHub repository.

**How**: `git clone https://github.com/{USERNAME}/{REPO}.git`

**Why**: We clone fresh every time instead of caching repos because:
- Repos change between deployments
- Caching introduces staleness bugs
- Each container is isolated and ephemeral

### Step 7: Build Server → Checkout Branch

**What happens**: Switches to the specified branch.

**How**: `git checkout {BRANCH}`

**Why**: Users can deploy from any branch — `main`, `dev`, `feature/new-ui`, etc.

### Step 8: Build Server → Inject Environment Variables

**What happens**: If the user passed custom environment variables, they're written to a `.env` file in the project root.

**How**: `echo "$ENV_VARS" | jq -r '.[] | "\(.key)=\(.value)"' > ".env"` followed by `export $(grep -v '^#' ".env" | xargs)`

**Why**: Many projects need `VITE_API_URL`, `NEXT_PUBLIC_API_KEY`, or other env vars at build time. Without injecting them, the build would use defaults (or fail).

### Step 9: Build Server → Detect Build Strategy

**What happens**: `script.js` analyzes the cloned project to determine what kind of project it is.

**How**: The `detectBuildStrategy()` function checks for:
- `pom.xml` → Java Maven project
- `build.gradle` → Java Gradle project
- `next.config.js` → Next.js project
- `vite.config.js` → Vite project
- `package.json` with `build` script → Generic Node project
- None of the above → Static HTML/CSS/JS project

**Why**: Different projects need different build commands. A Vite project needs `npm run build`, a Maven project needs `mvn clean install`, and a static site needs nothing.

### Step 10: Build Server → Install Dependencies

**What happens**: `npm install` (or equivalent) installs project dependencies.

**How**: Determined by `getBuildCommand()` — runs `npm install && npm run build` for Node projects.

**Why**: Dependencies are not included in Git repositories (they're in `.gitignore`). You must install them before building.

### Step 11: Build Server → Build Production Bundle

**What happens**: `npm run build` creates an optimized production build.

**How**: For Vite projects, this creates a `dist/` folder. For CRA projects, a `build/` folder. For Java, a `target/` folder.

**Why**: Development code includes source maps, hot reload, and debug logging. Production builds are minified, tree-shaken, and optimized for performance.

### Step 12: Build Server → Log Every Step to Kafka

**What happens**: Throughout the entire process, every action is logged to Kafka.

**How**: `publishLog("Installing dependencies...", "INFO")` sends a JSON message to the `build-logs` Kafka topic.

**Why**: Real-time visibility. The user can see exactly what's happening in their deployment, just like Vercel's build logs.

### Step 13: Build Server → Upload Build Output to S3

**What happens**: All files in the build output directory are uploaded to AWS S3.

**How**: The `uploadFolderToS3()` function:
1. Recursively walks the output directory (skipping `.git`, `node_modules`, `.env`)
2. Uploads files in batches of 5 (parallel within each batch)
3. Uses `PutObjectCommand` with correct MIME types
4. Stores files at path: `__outputs/{deploymentId}/{relative-path}`

**Why**: S3 provides durable, scalable, and cheap static file hosting. Files are served globally with low latency. The batch upload prevents overwhelming the S3 API with too many concurrent requests.

### Step 14: S3 → Static Website Live

**What happens**: The uploaded files are now accessible via an S3 URL.

**How**: The deployment URL follows the pattern: `https://{s3-bucket}.s3.{region}.amazonaws.com/__outputs/{deploymentId}/index.html`

**Why**: S3 static website hosting serves files directly to browsers. No server needed. It scales automatically to handle any amount of traffic.

### Step 15: Build Server → Save Deployment Metadata

**What happens**: After successful upload, the frontend calls `POST /api/deploy/create` to save deployment metadata to MongoDB.

**How**: Creates/updates a `Deployment` document with `{deploymentId, repoName, branch, username, url}`.

**Why**: MongoDB stores the persistent record of all deployments. Users can revisit their Dashboard to see all past deployments with their live URLs.

### Step 16: Cleanup

**What happens**: The build container exits after completion. The local build folder is cleaned up inside the container.

**How**: `fs.rmSync(folderToUpload, { recursive: true, force: true })` removes build artifacts before exit. The container itself stops and can be removed.

**Why**: Disk space is finite. Without cleanup, build artifacts would accumulate and eventually fill the disk.

---
---

# SECTION 6: INFRASTRUCTURE

## Infrastructure Overview

Vortex runs on AWS with the following infrastructure components, all provisioned via Terraform:

```
AWS Cloud
├── VPC (10.0.0.0/16)
│   ├── Public Subnet (10.0.1.0/24)
│   │   └── EC2 Instance (t3.large)
│   │       ├── Elastic IP (static)
│   │       ├── 30GB gp3 EBS (encrypted)
│   │       └── IAM Instance Profile (SSM access)
│   ├── Internet Gateway
│   ├── Route Table (0.0.0.0/0 → IGW)
│   └── Security Group (SSH, HTTP, HTTPS, app ports)
└── S3 Bucket (eshwar-vortex-storage)
```

## Why Single Instance?

For a personal/portfolio project, a single EC2 instance running all services via Docker Compose is the right trade-off:

| Factor | Multi-Instance | Single Instance (Vortex) |
|--------|---------------|-------------------------|
| **Cost** | $200+/month for separate instances | ~$60/month for one t3.large |
| **Complexity** | Service discovery, load balancers, networking | Simple Docker Compose networking |
| **Latency** | Network hops between services | All services on localhost/Docker bridge |
| **Reliability** | Higher (distributed) | Lower (single point of failure) |
| **Learning** | Harder to debug across instances | Easier to understand end-to-end |

**In production improvement**: You'd split services across multiple instances, use ECS/EKS for orchestration, and add an Application Load Balancer.

## Network Architecture

The VPC is configured with a single public subnet. All services run on the EC2 instance and communicate via Docker's bridge network:

```
Internet
    │
    ▼
Internet Gateway (igw)
    │
    ▼
Route Table (0.0.0.0/0 → igw)
    │
    ▼
Public Subnet (10.0.1.0/24)
    │
    ▼
EC2 Instance
    │
    ├── Elastic IP (permanent static IP)
    ├── Security Group (firewall rules)
    └── Docker Bridge Network ("deployment")
        ├── kafka-1, kafka-2, kafka-3
        ├── clickhouse, mongodb
        ├── vortex-app, vortex-nginx
        └── kafka-console
```

---
---

# SECTION 7: TERRAFORM

## What is Terraform?

Terraform is an open-source **Infrastructure as Code (IaC)** tool by HashiCorp that lets you define, provision, and manage cloud infrastructure using declarative configuration files written in HCL (HashiCorp Configuration Language).

Instead of clicking through the AWS Console to create an EC2 instance, VPC, security groups, etc., you write `.tf` files that describe your desired infrastructure state, and Terraform figures out how to create/modify/destroy resources to match that state.

## Why Terraform?

| Reason | Explanation |
|--------|-------------|
| **Reproducibility** | Run `terraform apply` to create identical infrastructure in any region, any account. No manual steps. |
| **Version Control** | Infrastructure changes are tracked in Git. You can review, revert, and audit changes. |
| **State Management** | Terraform tracks what it created. It knows what exists and what needs to change. |
| **Plan Before Apply** | `terraform plan` shows exactly what will be created/modified/destroyed before you commit. |
| **Multi-Provider** | Works with AWS, GCP, Azure, DigitalOcean, and 3000+ providers. |
| **Declarative** | You describe the end state, not the steps to get there. Terraform handles the ordering. |

## Why NOT CloudFormation?

| Factor | Terraform | CloudFormation |
|--------|-----------|----------------|
| **Multi-Cloud** | ✅ AWS, GCP, Azure, etc. | ❌ AWS only |
| **Language** | HCL (clean, readable) | JSON/YAML (verbose) |
| **State** | Explicit state file you control | Managed by AWS (opaque) |
| **Community** | Massive module registry | Limited to AWS solutions |
| **Drift Detection** | `terraform plan` | Drift detection is recent, limited |
| **Learning** | Industry standard for DevOps roles | Only relevant for AWS-specific roles |

## Why NOT AWS Console?

- **Not reproducible**: You can't easily recreate the same setup from clicks.
- **Not auditable**: No Git history of infrastructure changes.
- **Error-prone**: Manual clicks lead to misconfigurations.
- **Not scalable**: Managing 50+ resources manually is a nightmare.

## Core Terraform Concepts

### Provider

```hcl
provider "aws" {
  region = var.aws_region
}
```

A **provider** is a plugin that Terraform uses to interact with a cloud platform's API. The `aws` provider translates your HCL into AWS API calls. Vortex uses the `hashicorp/aws` provider version `~> 5.0`.

### Resources

Resources are the actual cloud objects that Terraform creates. In Vortex:

```hcl
resource "aws_instance" "vortex_host" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  ...
}
```

This creates an EC2 instance. The resource type (`aws_instance`) and name (`vortex_host`) together form a unique identifier.

### Variables

Variables allow you to parameterize your infrastructure:

```hcl
variable "instance_type" {
  description = "EC2 instance size type"
  type        = string
  default     = "t3.large"
}
```

Vortex defines variables for: `aws_region`, `instance_type`, `key_name`, `vpc_cidr`, `subnet_cidr`, `allowed_ssh_cidr`, and more. This means you can deploy Vortex in any region with any instance type by changing `terraform.tfvars`.

### Data Sources

Data sources let Terraform fetch information from existing resources:

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }
}
```

This dynamically finds the latest Ubuntu 24.04 AMI from Canonical. Instead of hardcoding an AMI ID (which changes per region and daily), Terraform queries AWS for the latest one.

### Outputs

Outputs display useful information after `terraform apply`:

```hcl
output "ssh_command" {
  value = "ssh -i ~/.ssh/${var.key_name}.pem ubuntu@${aws_eip.vortex_eip.public_ip}"
}
```

After provisioning, Terraform prints the exact SSH command you need to connect.

### State File

The `terraform.tfstate` file is Terraform's **source of truth** about what infrastructure exists. It maps your `.tf` files to real AWS resources (instance IDs, IP addresses, etc.).

**Critical**: If you lose the state file, Terraform doesn't know what it created. It will try to create everything again, causing conflicts.

**In Vortex**: The state file is stored locally (`infra/terraform/terraform.tfstate`). In production, you'd use a **remote backend** like S3 + DynamoDB for state locking.

### State Locking

When multiple people run `terraform apply` simultaneously, state locking prevents concurrent modifications. DynamoDB is commonly used for this with S3 backends.

### Terraform Plan

`terraform plan` is a dry run that shows what Terraform will do WITHOUT actually doing it:

```bash
$ terraform plan
+ aws_instance.vortex_host will be created
  + ami           = "ami-0abcd1234efgh5678"
  + instance_type = "t3.large"
  ...
```

The `+` means create, `~` means modify, `-` means destroy.

### Terraform Apply

`terraform apply` executes the plan and creates/modifies/destroys resources:

```bash
$ terraform apply
aws_vpc.main: Creating...
aws_vpc.main: Creation complete after 2s [id=vpc-0abc123]
aws_subnet.public: Creating...
...
Apply complete! Resources: 12 added, 0 changed, 0 destroyed.
```

### Terraform Destroy

`terraform destroy` tears down all infrastructure:

```bash
$ terraform destroy
aws_instance.vortex_host: Destroying... [id=i-0abc123]
aws_eip.vortex_eip: Destroying... [id=eipalloc-0abc123]
...
Destroy complete! Resources: 12 destroyed.
```

## How Vortex Provisions Infrastructure

Vortex's Terraform configuration creates these resources in order (Terraform handles dependency resolution automatically):

1. **VPC** (`networking.tf`) — Virtual private cloud with DNS support enabled
2. **Public Subnet** (`networking.tf`) — `10.0.1.0/24` in specified availability zone
3. **Internet Gateway** (`networking.tf`) — Allows outbound internet access
4. **Route Table** (`networking.tf`) — Routes `0.0.0.0/0` to the internet gateway
5. **Security Group** (`security.tf`) — 20+ rules for SSH, HTTP, HTTPS, Kafka, ClickHouse, MongoDB, etc.
6. **IAM Role** (`iam.tf`) — EC2 role with SSM policy for AWS Systems Manager access
7. **IAM Instance Profile** (`iam.tf`) — Attaches the role to the EC2 instance
8. **EC2 Instance** (`ec2.tf`) — Ubuntu 24.04, t3.large, 30GB gp3 encrypted EBS, with user_data bootstrap script
9. **Elastic IP** (`ec2.tf`) — Static public IP that persists across instance stop/start cycles
10. **EIP Association** (`ec2.tf`) — Binds the Elastic IP to the EC2 instance

### User Data Bootstrap Script

The `user_data.sh` script runs automatically when the EC2 instance first boots. It:

1. **Allocates 4GB swap space** — Protects against OOM during heavy Docker builds
2. **Installs prerequisites** — `ca-certificates`, `curl`, `gnupg`, `git`
3. **Adds Docker GPG key and repository** — For official Docker CE installation
4. **Installs Docker CE + Compose Plugin** — Full Docker engine with `docker compose`
5. **Adds `ubuntu` user to `docker` group** — No `sudo` needed for Docker commands
6. **Validates installations** — Verifies Docker, Docker Compose, and Git are working

Every step uses a `retry_cmd` function that retries failed commands up to 5 times with 5-second delays. This is crucial because `apt-get update` can fail on a freshly launched instance if the package mirrors are temporarily unavailable.

## Tradeoffs

| Decision | Trade-off |
|----------|-----------|
| **Local state file** | Simple but risky. No team collaboration, no state locking. |
| **Single AZ** | Lower cost but no availability zone redundancy. |
| **Public subnet only** | Simpler networking but databases are technically internet-accessible (protected by security groups). |
| **t3.large** | Enough RAM for Kafka + ClickHouse + MongoDB but costs ~$60/month. |
| **4GB swap** | Prevents OOM but adds I/O latency when swap is used. |

## Interview Questions

1. **What is the Terraform state file and why is it important?**
   - It's Terraform's record of what infrastructure exists. Without it, Terraform can't know what it created and would try to recreate everything, causing conflicts.

2. **What happens if you lose the state file?**
   - You'd need to import existing resources back into state using `terraform import`, or destroy everything and start fresh.

3. **What is the difference between `terraform plan` and `terraform apply`?**
   - `plan` is a dry run that shows what will change. `apply` actually executes the changes. Always plan before apply.

4. **How does Terraform handle dependencies between resources?**
   - Terraform builds a dependency graph from resource references. If `aws_subnet` references `aws_vpc.main.id`, Terraform knows to create the VPC first.

5. **What is `lifecycle { ignore_changes = [ami] }` doing in the EC2 resource?**
   - It prevents Terraform from replacing the instance when a new Ubuntu AMI is released. AMIs change daily; you don't want your server recreated every time.

6. **What is the purpose of the user_data script?**
   - It's a bootstrap script that runs on first boot. It installs Docker, Git, and other dependencies so the instance is ready for deployment.

7. **Why did you choose `gp3` over `gp2` for EBS volumes?**
   - gp3 provides 3,000 baseline IOPS and 125 MB/s throughput regardless of volume size, while gp2 scales IOPS with size. gp3 is also ~20% cheaper.

## Counter Questions

1. **"Why not use Pulumi instead of Terraform?"**
   - Pulumi uses general-purpose programming languages (Python, TypeScript), which is more flexible but introduces runtime complexity. Terraform's HCL is purpose-built for infrastructure definition — it's declarative, has no loops that could infinite-loop, and the community/module ecosystem is significantly larger.

2. **"Why not use Terraform Cloud?"**
   - For a personal project, local execution is simpler and free. Terraform Cloud adds remote execution, state management, and team features that are overkill for a single developer.

3. **"Why Elastic IP instead of just using the instance's public IP?"**
   - EC2 public IPs change when you stop/start an instance. Elastic IPs are static — your DNS records, Ansible inventory, and CI/CD pipelines don't break when you restart the server.

---
---

# SECTION 8: AWS

## EC2 (Elastic Compute Cloud)

### What is EC2?
EC2 is AWS's virtual server service. You get a virtual machine (called an "instance") running on AWS's physical hardware. You choose the OS, CPU, RAM, storage, and networking.

### Why EC2?
- **Full control**: You have root SSH access. Install anything. Configure anything.
- **Docker support**: Docker runs natively on EC2 instances.
- **Flexible sizing**: Start with `t3.micro` (free tier), scale to `m5.4xlarge` when needed.
- **Persistent**: Unlike Lambda/Fargate, EC2 instances persist. Long-running Docker containers (Kafka, MongoDB) need persistent processes.

### AMI (Amazon Machine Image)
An AMI is a template that contains the OS and pre-installed software. Vortex uses Ubuntu 24.04 (Noble Numbat) from Canonical (owner ID `099720109477`). The `data.aws_ami.ubuntu` data source dynamically finds the latest AMI.

### Instance Types
Vortex uses `t3.large` (2 vCPU, 8 GB RAM). Why?

| Service | Min RAM | Reason |
|---------|---------|--------|
| Kafka (×3 brokers) | ~768MB | Each broker configured with `-Xmx256M` heap |
| ClickHouse | ~1GB | Column store with compression buffers |
| MongoDB | ~512MB | WiredTiger cache |
| Application | ~256MB | Node.js + Express + Vite dev server |
| Build containers | ~512MB each | npm install + build processes |
| OS + Docker | ~512MB | Baseline overhead |

Total minimum: ~4GB. A `t3.large` with 8GB gives headroom for multiple concurrent builds and swap space.

### Security Groups
A security group is a **virtual firewall** that controls inbound and outbound traffic to the EC2 instance. Vortex defines granular rules:

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 22 | TCP | Admin CIDR only | SSH access |
| 80 | TCP | 0.0.0.0/0 | HTTP (Nginx) |
| 443 | TCP | 0.0.0.0/0 | HTTPS (future) |
| 3005 | TCP | Admin CIDR | Direct frontend access |
| 5005 | TCP | Admin CIDR | Direct backend API access |
| 8080 | TCP | Admin CIDR | Redpanda Console |
| 19092 | TCP | Admin CIDR | Kafka Broker 1 |
| 29092 | TCP | Admin CIDR | Kafka Broker 2 |
| 39092 | TCP | Admin CIDR | Kafka Broker 3 |
| 8123 | TCP | Admin CIDR | ClickHouse HTTP |
| 9000 | TCP | Admin CIDR | ClickHouse Native |
| 27017 | TCP | Admin CIDR | MongoDB |

**Important**: Database and broker ports are restricted to admin CIDR by default (a non-routable range `192.0.2.0/24`). This means they're blocked from the internet unless explicitly overridden. Only HTTP/HTTPS is open to the world.

VPC internal rules duplicate each port for the VPC CIDR (`10.0.0.0/16`) to allow inter-service communication within the VPC.

### IAM (Identity and Access Management)
Vortex creates an IAM role with the `AmazonSSMManagedInstanceCore` policy. This allows:
- **AWS Systems Manager** to manage the instance (patching, commands) without SSH.
- The instance to authenticate with other AWS services using instance metadata instead of hardcoded credentials.

### Elastic IP
A static public IP address that doesn't change when the instance stops/starts. This is critical because:
- Ansible inventory has a hardcoded IP (`13.235.26.67`)
- CI/CD pipelines SSH to a known IP
- DNS records (if configured) point to a static IP

### VPC (Virtual Private Cloud)
A logically isolated section of the AWS cloud. Vortex creates a VPC with CIDR `10.0.0.0/16`, giving 65,536 IP addresses.

### Subnet
A range of IP addresses within the VPC. Vortex uses a single public subnet (`10.0.1.0/24`) with 256 addresses. "Public" means instances get public IP addresses.

### Route Tables
Define where network traffic goes. The public route table has one route: `0.0.0.0/0 → Internet Gateway`, meaning all internet-bound traffic goes through the IGW.

### Internet Gateway
The gateway between the VPC and the public internet. Without it, the EC2 instance would have no internet access.

## S3 (Simple Storage Service)

Vortex uses S3 (`eshwar-vortex-storage` bucket) to store deployed website assets. Files are stored at:

```
__outputs/{deploymentId}/
├── index.html
├── assets/
│   ├── index-abc123.js
│   ├── index-def456.css
│   └── logo.svg
└── ...
```

S3 provides:
- **11 9's durability** (99.999999999%) — Your files won't disappear.
- **Unlimited storage** — No disk space management.
- **HTTP access** — Files are directly accessible via URLs.
- **Low cost** — ~$0.023/GB/month.

## Tradeoffs

| Decision | Pro | Con |
|----------|-----|-----|
| **EC2 over ECS** | Full control, Docker native | Manual scaling, no service discovery |
| **EC2 over Kubernetes** | Simple, no k8s overhead | No auto-healing, no rolling updates |
| **Single AZ** | Lower cost | No HA if AZ goes down |
| **Public subnet for everything** | Simple networking | Databases technically internet-facing |
| **gp3 EBS** | Consistent IOPS, cheaper | Not as fast as io2 for heavy workloads |

## Production Improvements

1. **Auto Scaling Group** — Automatically launch more instances under high load.
2. **Application Load Balancer** — Distribute traffic across multiple instances.
3. **Private subnets** — Put databases in private subnets with NAT Gateway for internet access.
4. **CloudWatch Alarms** — Alert on CPU > 80%, disk > 90%, etc.
5. **CloudFront CDN** — Serve S3 assets from edge locations globally.
6. **S3 Lifecycle Policies** — Auto-delete old deployment artifacts after 90 days.
7. **Multi-AZ** — Deploy across 2+ availability zones for high availability.

---
---

# SECTION 9: DOCKER

## What is Docker?

Docker is a platform for building, shipping, and running applications in **containers**. A container is a lightweight, standalone executable package that includes everything needed to run the application: code, runtime, libraries, and system tools.

## Why Docker?

| Problem Without Docker | Solution With Docker |
|----------------------|---------------------|
| "Works on my machine" — different OS, Node version, library versions | Consistent environment everywhere — same Dockerfile = same container |
| Installing 10 different services manually on a server | `docker compose up` — one command starts everything |
| Service conflicts (MongoDB version clash with ClickHouse) | Each service runs in its own isolated container |
| Security risks from running user code directly on the host | User builds run in ephemeral containers — no host access |
| Difficult cleanup — uninstalling a service leaves config files | `docker compose down` — everything is removed cleanly |

## Container vs VM

| Feature | Container | Virtual Machine |
|---------|-----------|-----------------|
| **Isolation** | Process-level (shared kernel) | Hardware-level (own kernel) |
| **Boot time** | Milliseconds | Minutes |
| **Size** | 10-500 MB | 1-10 GB |
| **Performance** | Near-native | 5-10% overhead (hypervisor) |
| **Density** | 100s per host | 10-20 per host |
| **Use case** | Microservices, CI/CD, builds | Full OS environments, legacy apps |

## How Docker Works Internally

### Namespaces

Linux namespaces provide **isolation**. Each container gets its own:

| Namespace | Isolates |
|-----------|----------|
| **PID** | Process IDs — container sees its processes as PID 1, 2, 3... |
| **NET** | Network interfaces — container gets its own IP, ports, routing |
| **MNT** | Filesystem mounts — container has its own root filesystem |
| **UTS** | Hostname — container can have its own hostname |
| **IPC** | Inter-process communication — shared memory is isolated |
| **USER** | User/group IDs — root inside container ≠ root on host |

### Cgroups (Control Groups)

Cgroups provide **resource limiting**. They control how much CPU, memory, disk I/O, and network a container can use.

In Vortex's Docker Compose, the Kafka brokers have memory limits:
```yaml
KAFKA_HEAP_OPTS: "-Xmx256M -Xms128M"
```

This caps each Kafka JVM to 256MB max heap, preventing one broker from consuming all system memory.

### Union File System (OverlayFS)

Docker images are built in **layers**. Each Dockerfile instruction creates a new layer:

```dockerfile
FROM node:18             # Layer 1: Base OS + Node.js
RUN apt-get install ...  # Layer 2: Additional packages
COPY package.json .      # Layer 3: Package manifest
RUN npm install          # Layer 4: Dependencies
COPY . .                 # Layer 5: Application code
```

Layers are **read-only** and **shared** between containers. If 10 containers use `node:18`, the base layer is stored once. Only writable top layers are unique per container.

### Docker Daemon

The Docker daemon (`dockerd`) is the background service that manages Docker objects (images, containers, networks, volumes). It listens on a Unix socket (`/var/run/docker.sock`).

**In Vortex**: The `vortex-app` container mounts the host's Docker socket:
```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

This is **Docker-in-Docker (DinD)** — the Vortex application container can create and manage other Docker containers (build server containers) on the host. This is how `deploy.controller.js` runs `docker run` from inside a container.

### Docker CLI vs Docker Engine

- **Docker CLI** (`docker`): The command-line tool you type commands into.
- **Docker Engine**: The daemon + API that actually does the work.
- **Docker Desktop**: The macOS/Windows GUI wrapper around Docker Engine.

## Vortex Dockerfiles

### Application Dockerfile (`/Dockerfile`)

```dockerfile
FROM node:18
RUN apt-get update && apt-get install -y docker.io  # For DinD
RUN npm install -g npm-run-all
WORKDIR /app
COPY frontend ./frontend
COPY backend ./backend
COPY package.json .
RUN npm install
RUN cd frontend && npm install
RUN cd backend && npm install
EXPOSE 3000 5000
CMD ["npm-run-all", "--parallel", "start-frontend", "start-backend"]
```

**Why install Docker inside the container?** Because the backend needs to run `docker run` to spawn build containers. Without the Docker CLI inside the app container, `deploy.controller.js` can't execute Docker commands.

**Why `npm-run-all --parallel`?** It starts both the frontend Vite dev server (port 3000) and the backend Express server (port 5000) simultaneously in a single container. This avoids needing two separate containers for frontend and backend.

### Build Server Dockerfile (`/build-server/Dockerfile`)

```dockerfile
FROM ubuntu:focal
RUN apt-get install -y curl git openjdk-11-jdk maven jq nodejs
WORKDIR /home/app
COPY main.sh script.js package*.json ./
RUN chmod +x main.sh script.js
ENTRYPOINT ["/home/app/main.sh"]
```

**Why Ubuntu instead of Alpine?** The build server needs to support multiple project types — Node.js, Java (Maven/Gradle), and plain HTML. Alpine uses `musl` instead of `glibc`, which causes compatibility issues with some npm packages and Java. Ubuntu provides maximum compatibility.

**Why include Java/Maven?** The build strategy detection supports Java projects (`pom.xml`, `build.gradle`). The build server is a universal builder, not just for Node.js.

### Nginx Dockerfile (`/nginx/Dockerfile`)

```dockerfile
FROM nginx:1.25-alpine
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Why Alpine for Nginx?** Nginx doesn't need `glibc` compatibility. Alpine is only 5MB, making the image tiny and fast to build.

**Why `daemon off;`?** Docker expects the main process to run in the foreground. If Nginx daemonizes, the container thinks the process exited and stops. `daemon off` keeps Nginx in the foreground.

## Docker Volumes

Volumes provide persistent storage that survives container restarts:

```yaml
volumes:
  - kafka1_data:/var/lib/kafka/data
  - kafka2_data:/var/lib/kafka/data
  - kafka3_data:/var/lib/kafka/data
  - mongo_data:/data/db
```

**Why volumes?** Without volumes, all data inside a container is lost when it restarts. Kafka messages, MongoDB documents, and ClickHouse tables would be wiped on every `docker compose down`.

## Docker Networks

Vortex uses a custom bridge network called `deployment`:

```yaml
networks:
  deployment:
    name: deployment
    driver: bridge
```

### Bridge Network

A bridge network creates an isolated network namespace where containers can communicate by name (Docker DNS). Without it, containers would need to use IP addresses (which change on restart).

**Why named `deployment`?** Build server containers need to join this network to reach Kafka brokers. By naming it `deployment` (not auto-generated), the `docker run` command can use `--network deployment`.

## What Happens If Docker Crashes?

| Scenario | Consequence | Mitigation |
|----------|------------|------------|
| **Docker daemon crashes** | All containers stop | `restart: unless-stopped` policy auto-restarts containers when daemon recovers |
| **OOM Killer kills a container** | One service goes down | Swap space (4GB) provides buffer; `restart: unless-stopped` restarts the container |
| **Container filesystem full** | Writes fail | 30GB EBS volume with monitoring |
| **Docker socket unavailable** | Build server can't spawn | Health checks detect; CD pipeline collects diagnostic logs |

## Memory Limits & CPU Limits

In Docker Compose, you can set resource limits:

```yaml
deploy:
  resources:
    limits:
      memory: 512M
      cpus: '0.5'
```

Vortex uses JVM-level memory limits for Kafka (`-Xmx256M`) instead of Docker-level limits. This is because JVM-based applications manage their own memory and Docker limits would cause abrupt OOM kills instead of graceful garbage collection.

## OOM Killer

The Linux OOM (Out of Memory) Killer is the kernel's last resort when the system runs out of memory. It selects a process to kill based on an OOM score (higher score = more likely to be killed).

**In Vortex**: The `user_data.sh` script allocates 4GB swap space to prevent OOM:
```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

Swap provides overflow memory that's slower (disk-based) but prevents the OOM Killer from terminating Docker containers.

## Multi-Stage Builds

Vortex doesn't use multi-stage builds currently, but in production you would:

```dockerfile
# Stage 1: Build
FROM node:18 AS builder
COPY . .
RUN npm ci && npm run build

# Stage 2: Production
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

This reduces the final image from ~1GB to ~25MB by excluding source code, `node_modules`, and build tools.

## Layer Caching

Docker caches each layer. If a layer hasn't changed, it's reused from cache:

```dockerfile
COPY package.json .      # Changes rarely
RUN npm install          # Cached if package.json unchanged
COPY . .                 # Changes every build
```

**Best practice**: Copy `package.json` first, then `npm install`, then copy the rest. This way, `npm install` only re-runs when dependencies change, not when your code changes.

## Interview Questions

1. **What is the difference between a Docker image and a Docker container?**
   - An image is a read-only template (like a class). A container is a running instance of an image (like an object). Multiple containers can share one image.

2. **How does Docker networking work?**
   - Docker creates virtual network bridges. Containers on the same bridge network can communicate using container names (Docker DNS resolves names to IPs).

3. **What is Docker-in-Docker and why does Vortex use it?**
   - DinD allows a container to create other containers by mounting the host's Docker socket. Vortex uses it so the backend can spawn isolated build containers.

4. **What is the `restart: unless-stopped` policy?**
   - The container restarts automatically unless it was explicitly stopped by the user. Covers crashes, OOM kills, and Docker daemon restarts.

5. **Why use named volumes instead of bind mounts?**
   - Named volumes are managed by Docker, portable, and work on any OS. Bind mounts depend on the host's directory structure.

## Counter Questions

1. **"Why not use Podman instead of Docker?"**
   - Podman is daemonless and rootless, which is more secure. However, Docker has broader ecosystem support, more documentation, and the Docker Compose format is industry standard. Vortex uses Docker-in-Docker via socket mounting, which is simpler with Docker.

2. **"Why not use Docker Swarm for orchestration?"**
   - Docker Swarm is dead. Kubernetes won the orchestration war. For a single-node setup, Docker Compose is simpler and more appropriate. Swarm would add complexity without benefit.

---
---

# SECTION 10: DOCKER COMPOSE

## What is Docker Compose?

Docker Compose is a tool for defining and running multi-container Docker applications. You use a `docker-compose.yml` file to configure all your services, networks, and volumes, then start everything with `docker compose up`.

## Why Docker Compose?

| Without Compose | With Compose |
|----------------|-------------|
| Run 8+ separate `docker run` commands with flags | `docker compose up -d` — one command |
| Manually create networks and volumes | Defined declaratively in YAML |
| No dependency management between services | `depends_on` with health check conditions |
| No service discovery | Container names resolve via Docker DNS |
| Hard to reproduce | `docker-compose.yml` is version-controlled |

## Vortex Docker Compose Deep Dive

### Service Dependency Chain

```
topic-init ──depends──> kafka-1, kafka-2, kafka-3
console ────depends──> kafka-1, kafka-2, kafka-3
application ─depends──> kafka-1 (started), clickhouse (healthy), mongodb (healthy)
nginx ──────depends──> application (healthy)
```

This ensures:
1. Kafka brokers start first
2. ClickHouse and MongoDB start and become healthy
3. The application starts after databases are ready
4. Nginx starts only after the application's health check passes

### Health Checks

```yaml
clickhouse:
  healthcheck:
    test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8123/ping"]
    interval: 10s
    timeout: 5s
    retries: 5

mongodb:
  healthcheck:
    test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
    interval: 10s
    timeout: 5s
    retries: 5

application:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
    interval: 15s
    timeout: 5s
    retries: 3
```

**Why health checks?** Without them, Docker considers a container "started" as soon as the process launches — even if MongoDB hasn't finished initializing its journal or ClickHouse hasn't loaded its tables. Health checks ensure the service is actually **ready to serve requests**.

### `depends_on` with Conditions

```yaml
application:
  depends_on:
    kafka-1:
      condition: service_started
    clickhouse:
      condition: service_healthy
    mongodb:
      condition: service_healthy
```

- `service_started` — Just wait for the container to start (Kafka has no built-in health check)
- `service_healthy` — Wait for the health check to pass (ClickHouse/MongoDB must be ready)

### Why Build Server Is NOT in Docker Compose

The build server is spawned **dynamically** per deployment using `docker run`. It's not a long-running service — it's an ephemeral container that:
1. Starts when a user deploys
2. Clones, builds, uploads
3. Exits when done

Including it in Docker Compose would mean it runs forever, which wastes resources and doesn't support concurrent builds for different users.

## Interview Questions

1. **What is the difference between `docker compose up` and `docker compose up --build`?**
   - `up` starts containers from existing images. `up --build` rebuilds images before starting. Use `--build` when Dockerfiles or build contexts change.

2. **What is `docker compose down --remove-orphans`?**
   - Stops and removes containers, networks. `--remove-orphans` also removes containers for services that are no longer defined in the compose file.

3. **How does Docker DNS resolution work in Compose?**
   - Docker creates an embedded DNS server for each network. Containers can reach each other by service name (e.g., `kafka-1:19092` resolves to the kafka-1 container's IP).

---
---

# SECTION 11: NGINX

## What is Nginx?

Nginx (pronounced "engine-x") is a high-performance HTTP server, reverse proxy, and load balancer. In Vortex, it serves as the **edge gateway** — the single entry point for all external traffic.

## Nginx Roles in Vortex

### 1. Reverse Proxy

```nginx
location /api/ {
    proxy_pass http://backend_nodes;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

Nginx forwards `/api/*` requests to the Express backend (`application:5000`). The client never communicates directly with Express.

**Why?** 
- **Security**: Express doesn't need to handle raw internet traffic.
- **Headers**: Nginx adds `X-Real-IP` and `X-Forwarded-For` so Express knows the real client IP (not Nginx's IP).
- **WebSocket support**: `proxy_http_version 1.1` and `Upgrade` headers enable WebSocket connections.

### 2. Load Balancer

```nginx
upstream backend_nodes {
    server application:5000;
}
```

Currently Vortex has one backend instance. But the `upstream` block is ready for horizontal scaling — add more servers and Nginx round-robins between them.

### 3. Gzip Compression

```nginx
gzip on;
gzip_comp_level 6;
gzip_types text/plain text/css application/json application/javascript;
```

Compresses responses before sending them to clients. A 500KB JavaScript bundle compresses to ~100KB, saving 80% bandwidth.

### 4. Security Headers

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "no-referrer-when-downgrade" always;
```

| Header | Protection |
|--------|-----------|
| `X-Frame-Options` | Prevents clickjacking (your site can't be embedded in iframes) |
| `X-XSS-Protection` | Enables browser's built-in XSS filter |
| `X-Content-Type-Options` | Prevents MIME type sniffing attacks |
| `Referrer-Policy` | Controls how much referrer info is sent with requests |

### 5. Health Check Proxy

```nginx
location /health {
    proxy_pass http://backend_nodes/health;
}
```

Exposes the backend's health endpoint through Nginx. The CD pipeline hits `http://{EC2_IP}/health` — this goes through Nginx to Express, which checks MongoDB status.

### 6. Kafka Console Proxy

```nginx
location /console/ {
    proxy_pass http://console_nodes/;
}
```

Provides access to the Redpanda Kafka monitoring console at `/console/`.

## Why Nginx? Why Not Expose Express Directly?

| Factor | Express Directly | Nginx + Express |
|--------|-----------------|----------------|
| **Performance** | Node.js is single-threaded, struggles with many static files | Nginx handles 10,000+ concurrent connections with event-driven architecture |
| **Security** | No built-in security headers, limited rate limiting | Security headers, SSL termination, request filtering |
| **Compression** | Gzip in Express uses CPU on the app thread | Nginx handles compression efficiently with worker processes |
| **Static files** | Express `express.static` is slow for many files | Nginx is purpose-built for static file serving |
| **SSL** | Need to manage certificates in Node.js | Nginx handles SSL termination with Let's Encrypt |
| **Crash recovery** | If Express crashes, the port is unavailable | Nginx can show a maintenance page while Express restarts |

## Interview Questions

1. **What is a reverse proxy and why is it important?**
   - A reverse proxy sits in front of backend servers and forwards client requests. It provides load balancing, SSL termination, caching, security, and hides internal infrastructure.

2. **What is the difference between `proxy_pass` and redirect?**
   - `proxy_pass` forwards the request internally (the client doesn't know). A redirect sends the client a 301/302 response telling them to make a new request to a different URL.

3. **Why `worker_processes auto;`?**
   - It automatically sets the number of Nginx worker processes to match the number of CPU cores. Each worker handles connections independently.

4. **What is `keepalive_timeout 65;`?**
   - It keeps the TCP connection open for 65 seconds after a request. Subsequent requests from the same client reuse the connection, avoiding TCP handshake overhead.

## Counter Questions

1. **"Why not use Traefik or HAProxy?"**
   - Traefik is excellent for dynamic container environments (auto-discovers services), but overkill for a static Compose setup. HAProxy is better for TCP load balancing. Nginx is the most common reverse proxy with the largest documentation ecosystem, making it the best learning choice.

2. **"Why not use an AWS Application Load Balancer?"**
   - ALB costs ~$20/month minimum. Nginx is free and runs on the same instance. For a single-instance deployment, Nginx provides the same features without additional AWS costs.

---
---

# SECTION 12: BACKEND

## Technology Choice: Express.js 5

### Why Express?

| Factor | Reason |
|--------|--------|
| **Minimal** | Express is a thin wrapper around Node.js HTTP. No magic, no abstractions. |
| **Ecosystem** | The most npm packages are built for Express. Middleware for anything. |
| **Control** | You decide the architecture. No imposed file structure like NestJS. |
| **Performance** | Low overhead. Express adds ~1ms to request processing. |
| **Community** | Most tutorials, Stack Overflow answers, and blog posts target Express. |

### Why Express over FastAPI?

| Factor | Express | FastAPI |
|--------|---------|---------|
| **Language** | JavaScript (same as frontend) | Python |
| **Performance** | Fast enough for most use cases | Faster for async I/O (uvloop) |
| **Type Safety** | Optional (TypeScript) | Built-in (Pydantic) |
| **Full-Stack** | Same language frontend+backend | Different language |

Vortex uses JavaScript on both frontend and backend. One language means less context switching, shared data structures, and easier hiring.

### Why Express over NestJS?

| Factor | Express | NestJS |
|--------|---------|--------|
| **Learning Curve** | Minimal — it's just middleware + routes | Steep — decorators, modules, DI, pipes, guards |
| **Overhead** | None — bare metal Express | Significant — TypeScript compilation, decorators, reflection |
| **Flexibility** | Full control | Opinionated structure |
| **Use Case** | APIs with <20 routes | Enterprise apps with 100+ routes |

Vortex has 5 route files and 5 controllers. NestJS would be overengineering.

## Backend Architecture

### Entry Point (`server.js`)

```
server.js
├── Security: helmet(), CORS, rate limiting
├── Health: GET /health → MongoDB status
├── Routes:
│   ├── /api/auth → auth.routes.js → auth.controller.js
│   ├── /api/github → git.routes.js → git.controller.js
│   ├── /api/logs → log.routes.js → log.controller.js
│   ├── /api/deploy → deploy.route.js → deploy.controller.js
│   └── /api/user → user.routes.js → user.controller.js
├── Error Handler: Centralized 500 handler
└── Graceful Shutdown: SIGTERM/SIGINT handlers
```

### Security Middlewares

1. **Helmet**: Sets 11+ HTTP security headers automatically.
2. **CORS**: Controls which origins can call the API. Configured via `CORS_ORIGIN` env var.
3. **Rate Limiting**: 5000 requests per 15 minutes per IP. High limit because the frontend polls for build logs.
4. **Trust Proxy**: `app.set('trust proxy', 1)` — tells Express to trust the first proxy (Nginx) for `X-Forwarded-For` headers.

### Graceful Shutdown

```javascript
const gracefulShutdown = async (signal) => {
    console.log(`Received ${signal}. Shutting down gracefully...`);
    server.close(async () => {
        await mongoose.connection.close();
        process.exit(0);
    });
};
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

**Why?** When Docker stops a container, it sends `SIGTERM`. Without graceful shutdown, in-flight requests would be dropped and database connections would leak.

### Data Models

**User Model**:
```javascript
{
    username: String (required, unique),
    fullname: String (required),
    email: String (required, unique),
    githubProfile: String (required),
    password: String,
    timestamps: true  // createdAt, updatedAt
}
```

**Deployment Model**:
```javascript
{
    deploymentId: String (required, unique),
    repoName: String (required),
    branch: String (required),
    username: String (required),
    logs: [{ message: String, level: String, timestamp: Date }],
    url: String (required),
    timestamps: true
}
```

### API Endpoints

| Method | Path | Controller | Purpose |
|--------|------|-----------|---------|
| `POST` | `/api/auth/register` | `registerUser` | Create new user account |
| `POST` | `/api/auth/login` | `loginUser` | Authenticate and get JWT |
| `POST` | `/api/github/repos` | `getRepos` | List user's GitHub repos |
| `GET` | `/api/github/branches/:owner/:repo` | `getRepoBranches` | List branches of a repo |
| `POST` | `/api/deploy/start` | `deployProject` | Trigger a new deployment |
| `POST` | `/api/deploy/create` | `createDeployment` | Save deployment metadata |
| `GET` | `/api/deploy/get` | `getDeploymentByRepoAndUser` | Check if deployment exists |
| `GET` | `/api/deploy/getdeploy` | `getDeploymentsByUser` | List all user deployments |
| `GET` | `/api/logs/:id` | `getLogsByDeploymentId` | Fetch build logs from ClickHouse |
| `GET` | `/api/user/getuser/:username` | `getUserByUsername` | Get user profile |
| `PUT` | `/api/user/updateuser/:username` | `updateUser` | Update user profile |
| `GET` | `/health` | inline | Health check with MongoDB status |

---
---

# SECTION 13: FRONTEND

## Technology Choice: React 19 + Vite + TailwindCSS 4

### Why React?

| Factor | React | Angular | Vue |
|--------|-------|---------|-----|
| **Learning Curve** | Moderate | Steep (TypeScript, RxJS, DI) | Gentle |
| **Ecosystem** | Largest (npm packages, tutorials) | Large but self-contained | Growing |
| **Performance** | Virtual DOM, concurrent rendering | Change detection cycles | Virtual DOM |
| **Job Market** | Most in-demand | Strong in enterprise | Growing |
| **Flexibility** | Choose your own tools | Full framework (opinionated) | Middle ground |
| **Community** | Massive | Large | Growing |

React was chosen because it has the largest ecosystem, most job postings, and works perfectly with Vite for fast development.

### Why Vite over Create React App (CRA)?

| Factor | Vite | CRA |
|--------|------|-----|
| **Dev Server Startup** | ~300ms (ES modules) | ~10s (webpack bundle) |
| **Hot Module Replacement** | Instant (< 50ms) | Slow (full reload sometimes) |
| **Build Speed** | Fast (Rollup + esbuild) | Slow (webpack) |
| **Configuration** | Minimal, plugin-based | Hidden webpack config (ejection needed) |
| **Status** | Actively maintained | Deprecated by React team |

### Frontend Architecture

```
src/
├── App.jsx                     # Root component with routing
├── main.jsx                    # Entry point with Redux + PersistGate
├── index.css                   # TailwindCSS import
├── components/
│   ├── Header.jsx              # Navigation bar
│   ├── Footer.jsx              # Footer
│   ├── PrivateRoute.jsx        # Auth guard (redirects to /auth if not logged in)
│   └── AutoLogout.jsx          # Auto-logout after 7 days
├── pages/
│   ├── WelcomePage.jsx         # Landing page (unauthenticated users)
│   ├── Auth.jsx                # Login/Register forms
│   ├── Home.jsx                # GitHub repo listing
│   ├── Dashboard.jsx           # Deployment history
│   ├── Deploy.jsx              # Deployment interface with live logs
│   ├── AccountSettings.jsx     # Profile settings
│   └── FallBack.jsx            # 404 page
├── redux/
│   ├── store.js                # Redux store with persist config
│   └── user/
│       └── UserSlice.js        # Auth state (user, token, loginTime)
└── services/
    └── api.js                  # Axios instance with base URL
```

### State Management: Redux Toolkit + Redux Persist

**Why Redux Toolkit?**
- Global auth state (user, token) needed across all pages
- `createSlice` reduces boilerplate (no action constants, no switch statements)
- Built-in Immer for immutable state updates

**Why Redux Persist?**
- User session survives page refresh (token stored in localStorage)
- `whitelist: ['user']` — only persist the user slice, not transient state
- `PersistGate` delays rendering until state is rehydrated from storage

### User Slice State

```javascript
{
    user: null | { username, fullname, email, githubProfile },
    token: null | "jwt-token-string",
    loading: false | true,
    error: null | "error message",
    loginTime: null | timestamp
}
```

**Actions**: `loginStart`, `loginSuccess`, `loginFailure`, `logoutSuccess`, `logout`, `resetUser`

### Auto-Logout Component

The `AutoLogout` component checks if the current time exceeds `loginTime + 7 days`. If so, it dispatches `logout()` and redirects to `/auth`. This runs on every page load.

### Routing

| Path | Component | Protected? | Description |
|------|-----------|-----------|-------------|
| `/` | `WelcomePage` | No | Landing page (redirects to `/home` if logged in) |
| `/auth` | `Auth` | No | Login/Register |
| `/home` | `Home` | Yes | GitHub repo listing |
| `/dashboard` | `Dashboard` | Yes | Deployment history |
| `/deploy/:username/:repo` | `Deploy` | Yes | Deployment interface |
| `/account-settings` | `AccountSettings` | Yes | Profile management |
| `*` | `FallBack` | Yes | 404 page |

### Vite Proxy Configuration

```javascript
proxy: {
    '/api': {
        target: process.env.VITE_API_TARGET || 'http://localhost:5000',
        secure: false,
        changeOrigin: true,
    }
}
```

During development, Vite proxies `/api` requests to the Express backend. This avoids CORS issues in development.

---
---

# SECTION 14: MONGODB

## What is MongoDB?

MongoDB is a **document database** (NoSQL) that stores data as JSON-like documents (BSON). Unlike relational databases with rows and columns, MongoDB stores flexible, schema-less documents in collections.

## Why MongoDB?

| Factor | MongoDB | PostgreSQL |
|--------|---------|-----------|
| **Schema** | Flexible (no migrations) | Rigid (ALTER TABLE for changes) |
| **Data Model** | Documents (JSON-like) | Rows and columns |
| **Scaling** | Horizontal (sharding built-in) | Vertical primarily |
| **Performance** | Fast reads/writes for document access | Better for complex joins |
| **Use Case** | User profiles, metadata, logs | Financial data, relational data |
| **ORM** | Mongoose (elegant, schema validation) | Sequelize, TypeORM (more boilerplate) |

### Why MongoDB for Vortex Specifically?

Vortex stores:
- **Users**: `{username, fullname, email, githubProfile, password}` — a natural document
- **Deployments**: `{deploymentId, repoName, branch, username, url, logs[]}` — nested arrays fit MongoDB's document model

There are **no joins** needed. You never need "all users who deployed repo X". Queries are always:
- "Get user by email" (login)
- "Get deployments by username" (dashboard)

These are simple key-based lookups that MongoDB handles faster than PostgreSQL because there's no SQL parsing overhead.

### Why NOT PostgreSQL?

| Scenario | Better Choice |
|----------|-------------|
| User profiles with nested data | MongoDB ✅ |
| Deployment metadata with variable schemas | MongoDB ✅ |
| Financial transactions with ACID guarantees | PostgreSQL ✅ |
| Complex multi-table joins | PostgreSQL ✅ |
| Analytics with GROUP BY | PostgreSQL ✅ (but ClickHouse even better) |

Vortex doesn't need joins, transactions, or complex queries. MongoDB's simplicity wins.

## How Vortex Uses MongoDB

### Collections

**users**:
```json
{
    "_id": "ObjectId(64abc...)",
    "username": "eshwarsai",
    "fullname": "Eshwar Sai",
    "email": "eshwar@example.com",
    "githubProfile": "Eshwarsai-07",
    "password": "$2a$10$hashed...",
    "createdAt": "2026-07-01T10:00:00Z",
    "updatedAt": "2026-07-01T10:00:00Z"
}
```

**deployments**:
```json
{
    "_id": "ObjectId(64def...)",
    "deploymentId": "dep-abc123-xyz",
    "repoName": "my-portfolio",
    "branch": "main",
    "username": "eshwarsai",
    "url": "https://eshwar-vortex-storage.s3.eu-north-1.amazonaws.com/__outputs/dep-abc123-xyz/index.html",
    "logs": [],
    "createdAt": "2026-07-15T14:30:00Z",
    "updatedAt": "2026-07-15T14:32:00Z"
}
```

### Indexes

Mongoose automatically creates indexes on `unique: true` fields:
- `users.username` — unique index
- `users.email` — unique index
- `deployments.deploymentId` — unique index

### Connection

```javascript
mongoose.connect(process.env.MONGO_URI || 'mongodb://localhost:27017/vortex')
```

Inside Docker Compose, `MONGO_URI=mongodb://mongodb:27017/vortex` — `mongodb` resolves to the MongoDB container's IP via Docker DNS.

## Interview Questions

1. **What is the difference between MongoDB and a relational database?**
   - MongoDB stores documents (flexible JSON), relational DBs store rows in tables. MongoDB doesn't require predefined schemas or joins.

2. **What is Mongoose and why use it?**
   - Mongoose is an ODM (Object Document Mapper) for MongoDB. It provides schema validation, middleware hooks, and a cleaner API than the raw MongoDB driver.

3. **What happens if MongoDB goes down in Vortex?**
   - New user registrations and logins fail. New deployments can't save metadata. Existing deployments' live URLs (on S3) continue working. Build logs (in ClickHouse) are unaffected.

4. **Why not use MongoDB for build logs?**
   - Build logs are append-heavy, time-series data. ClickHouse is optimized for this pattern with 10-100x better compression and query performance on analytical queries.

---
---

# SECTION 15: KAFKA

## What is Kafka?

Apache Kafka is a **distributed event streaming platform**. It's a highly scalable, fault-tolerant message queue that can handle millions of messages per second.

## Why Kafka?

| Requirement | Solution |
|-------------|----------|
| **Real-time log streaming** | Kafka produces/consumes messages with millisecond latency |
| **Decoupling** | Build server doesn't know about ClickHouse. It just sends to Kafka. |
| **Durability** | Messages are persisted to disk. If ClickHouse is temporarily down, no data loss. |
| **Scalability** | Can handle 10,000+ builds producing logs simultaneously |
| **Replay** | Consumers can re-read messages from any offset. Useful for debugging. |

## Why NOT RabbitMQ?

| Factor | Kafka | RabbitMQ |
|--------|-------|----------|
| **Throughput** | Millions of msg/sec | Thousands of msg/sec |
| **Persistence** | Messages persist on disk | Messages deleted after consumption |
| **Replay** | Consumers can re-read from any offset | Messages gone once consumed |
| **Use Case** | Event streaming, log aggregation | Task queues, request/reply |
| **Consumer Model** | Pull-based (consumers fetch) | Push-based (broker pushes) |

Vortex needs **log persistence and replay**, not task queuing. Kafka is the right tool.

## Why NOT Redis Streams?

| Factor | Kafka | Redis Streams |
|--------|-------|--------------|
| **Durability** | Disk-based, replication | Memory-first (can persist) |
| **Scalability** | Horizontally scalable cluster | Single-node or Redis Cluster |
| **Ecosystem** | ClickHouse native Kafka consumer | Requires custom consumers |
| **Maturity** | Industry standard for streaming | Newer, less battle-tested |

ClickHouse has a built-in **Kafka Engine** that directly consumes from Kafka topics. No custom consumer code needed. This integration doesn't exist for Redis Streams.

## How Kafka Works

### Architecture

```
┌──────────┐     ┌──────────────────────────┐     ┌──────────────┐
│ Producer │────▶│     Kafka Cluster        │────▶│   Consumer   │
│ (Build   │     │                          │     │ (ClickHouse) │
│  Server) │     │  ┌────────┐ ┌────────┐  │     │              │
│          │     │  │Broker 1│ │Broker 2│  │     │              │
│          │     │  │(Leader)│ │(Replica)│ │     │              │
│          │     │  └────────┘ └────────┘  │     │              │
│          │     │  ┌────────┐             │     │              │
│          │     │  │Broker 3│             │     │              │
│          │     │  │(Replica)│            │     │              │
│          │     │  └────────┘             │     │              │
│          │     └──────────────────────────┘     └──────────────┘
└──────────┘
```

### Key Concepts

**Producer**: The entity that sends messages to Kafka. In Vortex, the build server (`script.js`) is the producer — it sends build log messages to the `build-logs` topic.

**Consumer**: The entity that reads messages from Kafka. In Vortex, ClickHouse is the consumer — its Kafka Engine table automatically consumes from `build-logs`.

**Broker**: A Kafka server that stores and serves messages. Vortex runs 3 brokers for fault tolerance.

**Topic**: A named stream of messages. Vortex has one topic: `build-logs`.

**Partition**: A topic is divided into partitions for parallelism. Vortex uses 1 partition (sufficient for current scale).

**Replication**: Each partition's data is replicated across multiple brokers. Vortex uses replication factor 3 (configured via `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3`).

**Offset**: A sequential number assigned to each message in a partition. Consumers track their position using offsets, enabling replay.

**Consumer Group**: A group of consumers that share the workload. ClickHouse uses consumer group `clickhouse-consumer`. Each partition is consumed by exactly one consumer in the group.

### KRaft Mode (No ZooKeeper)

Vortex uses Kafka in **KRaft mode** (Kafka Raft). This is the modern consensus protocol that replaces ZooKeeper:

```yaml
KAFKA_PROCESS_ROLES: broker,controller
KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093
```

Each broker is both a broker AND a controller. The 3 brokers form a quorum — decisions require 2/3 agreement (majority).

**Why KRaft over ZooKeeper?**
- ZooKeeper is a separate service to manage (extra complexity)
- KRaft is built into Kafka (fewer moving parts)
- Faster controller elections
- Lower memory footprint

### How Vortex Uses Kafka

1. **Build server starts** → Kafka producer connects to `kafka-1:19092`
2. **During build** → `publishLog("Installing dependencies...", "INFO")` sends to topic `build-logs`
3. **Message format**: `{"deployment_id": "abc123", "log_message": "...", "log_level": "INFO"}`
4. **ClickHouse** continuously consumes from `build-logs` via its Kafka Engine table
5. **Materialized View** transforms and stores in the MergeTree table
6. **Frontend** polls `GET /api/logs/{id}` → queries ClickHouse for that deployment's logs

### Kafka Configuration Details

```yaml
KAFKA_HEAP_OPTS: "-Xmx256M -Xms128M"  # Memory limits per broker
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3  # Consumer offsets replicated 3x
KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2  # Min in-sync replicas
KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"  # Topics must be created explicitly
```

`AUTO_CREATE_TOPICS_ENABLE: false` is a best practice. Auto-created topics can have wrong partition counts and replication factors.

## Interview Questions

1. **What is Kafka and why is it used for log streaming?**
   - Kafka is a distributed event streaming platform that handles high-throughput message delivery with persistence. For logs, it provides real-time streaming with the ability to replay messages and scale horizontally.

2. **What is the difference between a topic and a partition?**
   - A topic is a logical channel for messages. A partition is a physical subdivision of a topic that enables parallelism. Messages within a partition are ordered.

3. **What happens if one Kafka broker goes down?**
   - With 3 brokers and replication factor 3, all data exists on all brokers. The remaining 2 brokers continue serving. Quorum requires 2/3, so the cluster remains operational.

4. **What is KRaft mode?**
   - KRaft (Kafka Raft) is Kafka's built-in consensus protocol that replaces ZooKeeper. Brokers elect a controller using Raft consensus without an external coordination service.

5. **How does ClickHouse consume from Kafka?**
   - ClickHouse has a native Kafka Engine that creates a consumer group and reads messages directly. A Materialized View transforms and stores messages into a MergeTree table.

---
---

# SECTION 16: CLICKHOUSE

## What is ClickHouse?

ClickHouse is an open-source **columnar database** designed for **Online Analytical Processing (OLAP)**. It stores data by columns instead of rows, enabling extreme compression and blazing-fast analytical queries.

## Why a Column Store?

Traditional row-based databases (MySQL, PostgreSQL) store data like:

```
Row 1: [deployment_id, log_message, log_level, timestamp]
Row 2: [deployment_id, log_message, log_level, timestamp]
```

ClickHouse stores data like:

```
Column: deployment_id → [abc123, abc123, abc123, def456, def456, ...]
Column: log_message   → ["npm install", "Building...", "Done", "Cloning", ...]
Column: log_level     → [INFO, INFO, INFO, INFO, INFO, ...]
Column: timestamp     → [2026-07-15 10:00:01, 10:00:02, 10:00:03, ...]
```

### Benefits:

1. **Compression**: Same-type data compresses much better. A column of `deployment_id` with repeated values compresses 10-50x.
2. **Query Speed**: `SELECT * WHERE deployment_id = 'abc123'` only reads the `deployment_id` column to find matching rows, then reads other columns only for those rows.
3. **I/O Efficiency**: Analytical queries typically read a few columns across millions of rows. Column store reads only needed columns.

## Why NOT MongoDB Aggregation for Logs?

| Factor | ClickHouse | MongoDB |
|--------|-----------|---------|
| **100K log rows query** | 5-50ms | 500-2000ms |
| **Compression** | 10-50x (LZ4/ZSTD) | 2-3x |
| **Insert throughput** | Millions/sec (batched) | Thousands/sec |
| **Kafka integration** | Native Kafka Engine | Requires custom consumer |
| **Use case** | Time-series analytics | Document storage |

Logs are append-only, time-ordered, and queried by `deployment_id` + `ORDER BY timestamp`. This is exactly what ClickHouse is optimized for.

## How Vortex Uses ClickHouse

### Database Schema

```sql
-- 1. Kafka Consumer Table
CREATE TABLE logs.log_queue (
    deployment_id String,
    log_message String,
    log_level String
) ENGINE = Kafka(
    'kafka-1:19092,kafka-2:19092,kafka-3:19092',
    'build-logs',
    'clickhouse-consumer',
    'JSONEachRow'
) SETTINGS kafka_skip_broken_messages = 1;
```

This table **IS** the Kafka consumer. It doesn't store data — it reads from the `build-logs` topic in real-time.

```sql
-- 2. Storage Table (MergeTree)
CREATE TABLE logs.build_logs (
    created_at DateTime64(3, 'Asia/Kolkata'),
    log_uuid UUID DEFAULT generateUUIDv4(),
    deployment_id String,
    log_message String,
    log_level String
) ENGINE = MergeTree
ORDER BY created_at;
```

MergeTree is ClickHouse's primary storage engine. It stores data in sorted, compressed parts on disk. `ORDER BY created_at` means logs are stored chronologically, making time-range queries extremely fast.

```sql
-- 3. Materialized View (auto-transform + insert)
CREATE MATERIALIZED VIEW logs.kafka_queue TO logs.build_logs
AS SELECT
    toTimezone(now(), 'Asia/Kolkata') AS created_at,
    generateUUIDv4() AS log_uuid,
    deployment_id,
    log_message,
    log_level
FROM logs.log_queue;
```

The Materialized View acts as a **trigger**: whenever a new message arrives in `log_queue` (from Kafka), it automatically adds a timestamp and UUID, then inserts into `build_logs`.

### Query Path

```
Frontend → GET /api/logs/{deploymentId}
    ↓
log.controller.js → ClickHouse query
    ↓
SELECT log_uuid, deployment_id, log_message, log_level, created_at
FROM build_logs
WHERE deployment_id = '{id}'
ORDER BY created_at ASC
```

## Why NOT PostgreSQL for Logs?

| Query | PostgreSQL | ClickHouse |
|-------|-----------|-----------|
| 1M rows, WHERE + ORDER BY | ~2 seconds | ~50ms |
| Disk usage for 1M logs | ~200MB | ~20MB (10x compression) |
| Insert throughput | ~10K/sec | ~1M/sec |

## Why NOT Elasticsearch for Logs?

| Factor | ClickHouse | Elasticsearch |
|--------|-----------|--------------|
| **Query speed** | Faster for structured queries | Better for full-text search |
| **Resource usage** | Low (runs on 512MB) | High (needs 4GB+ heap) |
| **Complexity** | SQL queries | JSON query DSL |
| **Use case** | Structured log analytics | Text search, log exploration |

Vortex logs are structured (deployment_id, message, level). No full-text search needed. ClickHouse is simpler and more resource-efficient.

## Interview Questions

1. **What is a columnar database and when should you use one?**
   - A columnar DB stores data by columns instead of rows. Use it when queries scan many rows but few columns (analytics, time-series, logs).

2. **What is a Materialized View in ClickHouse?**
   - It's a trigger that automatically transforms and inserts data when new rows arrive in the source table. It's how ClickHouse pipes data from Kafka to storage.

3. **What is the MergeTree engine?**
   - MergeTree is ClickHouse's primary storage engine. It stores data in sorted, compressed parts and periodically merges them for optimal query performance.

---
---

# SECTION 17: CI/CD

## What is CI/CD?

**CI (Continuous Integration)**: Automatically building, testing, and validating code every time changes are pushed. Catches bugs early.

**CD (Continuous Deployment)**: Automatically deploying validated code to production. Eliminates manual deployment steps.

## Vortex CI/CD Pipeline

```
Developer pushes to GitHub
         │
         ▼
    ┌─────────────────┐
    │  Pull Request    │─────▶ CI Pipeline (ci.yml)
    │  to main         │      ├── Checkout code
    │                  │      ├── Install + test backend
    │                  │      ├── Install + build frontend
    │                  │      ├── Validate Docker Compose
    │                  │      ├── Build Docker images
    │                  │      ├── Validate Terraform
    │                  │      ├── Trivy security scan
    │                  │      ├── Gitleaks secret scan
    │                  │      └── Snyk dependency scan
    └─────────────────┘
         │
         ▼ (PR merged)
    ┌─────────────────┐
    │  Push to main    │─────▶ CD Pipeline (cd.yml)
    │                  │      ├── Checkout code
    │                  │      ├── Setup SSH keys
    │                  │      ├── Verify SSH connection
    │                  │      ├── Run Ansible playbook
    │                  │      ├── Health check verification
    │                  │      └── Collect debug logs (on failure)
    └─────────────────┘
```

---
---

# SECTION 18: GITHUB ACTIONS

## What is GitHub Actions?

GitHub Actions is a CI/CD platform built into GitHub. It runs workflows (YAML files) in response to repository events (push, PR, manual trigger).

## Core Concepts

| Concept | Definition |
|---------|-----------|
| **Workflow** | A YAML file in `.github/workflows/` that defines the automation |
| **Job** | A set of steps that run on the same runner machine |
| **Step** | An individual task in a job (run a command, use an action) |
| **Runner** | The virtual machine that executes jobs (`ubuntu-latest`) |
| **Action** | A reusable unit of code (e.g., `actions/checkout@v4`) |
| **Secret** | Encrypted environment variable (e.g., `EC2_PRIVATE_KEY`) |
| **Artifact** | Files produced by a job (e.g., test reports, build output) |
| **Cache** | Stored dependencies to speed up future runs |

## Vortex CI Pipeline (`ci.yml`)

**Triggers**: Pull requests to `main`, manual dispatch

**Steps**:

1. **Checkout Code** — `actions/checkout@v4`
2. **Setup Node.js 20** — with npm cache from `backend/package-lock.json`
3. **Install Backend Dependencies** — `npm ci` (clean install from lockfile)
4. **Run Backend Tests** — `npm test` (continues if no tests configured)
5. **Install Frontend Dependencies** — `npm install`
6. **Build Frontend** — `npm run build` (catches TypeScript/bundler errors)
7. **Validate Docker Compose** — `docker compose -f services/docker-compose.yml config`
8. **Build Docker Images** — `docker compose build` (tests Dockerfiles without starting)
9. **Setup Terraform** — `hashicorp/setup-terraform@v3`
10. **Check Terraform Formatting** — `terraform fmt -check`
11. **Initialize Terraform** — `terraform init -backend=false`
12. **Validate Terraform** — `terraform validate`
13. **Trivy Filesystem Scan** — Checks for known vulnerabilities
14. **Gitleaks Secret Scanner** — Checks for exposed secrets in code/history
15. **Snyk Dependency Scan** — Checks npm packages for security issues

## Vortex CD Pipeline (`cd.yml`)

**Triggers**: Push/merge to `main`, manual dispatch

**Steps**:

1. **Checkout Repository**
2. **Setup SSH** — Write private key, add host to known_hosts
3. **Verify SSH Connection** — `ssh ... "echo OK"`
4. **Run Ansible Playbook** — `ansible-playbook -i inventory.ini playbook.yml`
5. **Health Check** — Curl health endpoint with retry loop (12 attempts, 15s apart = 3 min)
6. **Collect Debug Logs** (on failure) — SSH to EC2, get `docker ps`, `docker logs`, compose status

### Why GitHub Actions?

| Factor | GitHub Actions | Jenkins | GitLab CI |
|--------|---------------|---------|-----------|
| **Setup** | Zero (built into GitHub) | Install/maintain server | Separate GitLab instance |
| **Cost** | 2,000 min/month free | Server costs | Included with GitLab |
| **Integration** | Native GitHub (PRs, secrets) | Webhooks needed | Native GitLab |
| **Runners** | Managed by GitHub | Self-managed | Shared or self-managed |
| **Learning** | YAML + marketplace actions | Groovy pipelines | YAML |

GitHub Actions was chosen because:
- Zero infrastructure cost (free tier)
- Native GitHub integration (PR checks, branch protection)
- Marketplace actions for Terraform, Docker, security scanning

---
---

# SECTION 19: ANSIBLE

## What is Ansible?

Ansible is an **agentless** configuration management and automation tool. It connects to remote servers via SSH and executes tasks defined in YAML playbooks.

## Core Concepts

| Concept | Definition |
|---------|-----------|
| **Playbook** | YAML file defining tasks to execute on remote hosts |
| **Inventory** | File listing target hosts and their connection details |
| **Role** | Reusable set of tasks, handlers, templates, and variables |
| **Handler** | Task that runs only when notified (e.g., restart after config change) |
| **Template** | Jinja2 file with variable substitution (e.g., `.env.j2` → `.env`) |
| **Idempotency** | Running the same playbook twice produces the same result (safe to re-run) |

## Why Ansible?

| Factor | Ansible | Bash Scripts |
|--------|---------|-------------|
| **Idempotency** | Built-in (checks before acting) | You must code it yourself |
| **Error Handling** | Automatic with retries/assertions | Manual try/catch in bash |
| **Templates** | Jinja2 with variable substitution | Sed/awk replacements (fragile) |
| **Readability** | YAML is human-readable | Complex bash is hard to maintain |
| **Modules** | 3000+ built-in modules (git, docker, apt) | Everything is custom |

### Why NOT Bash Scripts?

Consider the difference:

**Bash**:
```bash
if [ ! -d "/home/ubuntu/vortex" ]; then
    git clone https://github.com/Eshwarsai-07/Vortex.git /home/ubuntu/vortex
fi
cd /home/ubuntu/vortex && git pull
```

**Ansible**:
```yaml
- name: Clone or pull the Vortex source code repository
  ansible.builtin.git:
    repo: "{{ repository_url }}"
    dest: "{{ project_directory }}"
    version: "{{ repository_branch }}"
    force: yes
```

The Ansible version handles: directory creation, clone vs pull, branch switching, error handling, and changed-state detection — all automatically.

## How Vortex Uses Ansible

### Playbook (`playbook.yml`)
```yaml
- name: Deploy Vortex Application Suite
  hosts: vortex_hosts
  become: true
  roles:
    - vortex
```

### Role Task Chain

```
main.yml
├── 1. repository.yml  → Clone/pull Vortex source code
├── 2. env.yml         → Generate .env from template
├── 3. deploy.yml      → Docker Compose build + start + build-server image
├── 4. flush_handlers   → Execute restarts if .env changed
└── 5. health.yml      → Verify containers healthy + health endpoint responding
```

### The `.env.j2` Template

Ansible uses Jinja2 templates to generate configuration files:

```
NODE_ENV={{ node_env }}
MONGO_URI=mongodb://localhost:27017/vortex
KAFKA_BROKER=kafka-1:19092
JWT_SECRET={{ jwt_secret }}
AWS_ACCESS_KEY_ID={{ aws_access_key_id }}
```

Variables like `{{ jwt_secret }}` are resolved from:
1. `defaults/main.yml` (lowest priority)
2. `group_vars/all.yml`
3. `--extra-vars` on command line (highest priority)
4. Environment variables via `lookup('env', 'AWS_ACCESS_KEY_ID')`

### Handler: Restart Docker Services

```yaml
- name: Restart Docker services
  ansible.builtin.shell:
    cmd: docker compose -f {{ docker_compose_file }} down && docker compose -f {{ docker_compose_file }} up -d
```

This handler only runs when the `.env` template changes (triggered by `notify: Restart Docker services` in `env.yml`).

## Interview Questions

1. **What is idempotency and why is it important?**
   - Idempotency means running the same operation multiple times produces the same result. If Ansible deploys successfully, running it again changes nothing. This makes deployments safe to retry.

2. **What is the difference between tasks and handlers?**
   - Tasks always run (in order). Handlers only run when notified by a task that made changes. This avoids unnecessary restarts.

3. **How does Ansible connect to remote servers?**
   - Via SSH. No agent needed on the remote server. The inventory file specifies the host IP, user, and SSH key.

---
---

# SECTION 20: SECURITY

## Security Measures in Vortex

### Application Layer

| Measure | Implementation | Purpose |
|---------|---------------|---------|
| **Helmet** | `app.use(helmet())` | 11+ security HTTP headers |
| **CORS** | Configured origin whitelist | Prevent unauthorized cross-origin requests |
| **Rate Limiting** | 5000 req/15min per IP | Prevent brute force and DoS |
| **Password Hashing** | bcrypt (10 salt rounds) | Protect passwords at rest |
| **JWT Authentication** | 7-day expiry tokens | Stateless session management |
| **Input Validation** | Controller-level field checks | Prevent missing/malformed data |
| **Graceful Shutdown** | SIGTERM/SIGINT handlers | Clean connection closure |

### Infrastructure Layer

| Measure | Implementation | Purpose |
|---------|---------------|---------|
| **Security Groups** | Port-specific CIDR rules | Network-level firewall |
| **SSH Key Auth** | RSA key pair (no passwords) | Secure remote access |
| **Encrypted EBS** | `encrypted = true` | Data-at-rest encryption |
| **IMDSv2** | `http_tokens = "required"` | Prevent SSRF attacks on instance metadata |
| **IAM Least Privilege** | Only SSM policy attached | Minimize blast radius |
| **Non-routable defaults** | `192.0.2.0/24` for admin ports | Database ports blocked by default |

### CI/CD Security

| Tool | Purpose |
|------|---------|
| **Trivy** | Filesystem vulnerability scanning |
| **Gitleaks** | Secret/credential leak detection in code and git history |
| **Snyk** | npm dependency vulnerability scanning |
| **GitHub Secrets** | Encrypted storage for API keys, SSH keys, tokens |

### Nginx Security Headers

```
X-Frame-Options: SAMEORIGIN          → Prevents clickjacking
X-XSS-Protection: 1; mode=block     → Enables XSS filter
X-Content-Type-Options: nosniff      → Prevents MIME sniffing
Referrer-Policy: no-referrer-when-downgrade → Controls referrer info
```

### Docker Security

- **Docker socket mounting**: Only the application container has access to the Docker socket.
- **Network isolation**: Each deployment runs in its own container with no access to other deployments.
- **No privileged mode**: Containers run without `--privileged`.
- **`.env` file permissions**: Ansible sets `mode: '0600'` (read/write for owner only).

---
---

# SECTION 21: FAILURE SCENARIOS

This is what interviewers love. Here's every failure scenario, what happens, and how Vortex handles it.

## MongoDB Down

**What happens**: Login fails. Registration fails. Dashboard can't load deployments.

**How it's detected**: Health endpoint returns `{"database": "DOWN"}`. CD pipeline health check fails.

**Impact**: Users can't authenticate or view history. BUT — existing deployed websites on S3 continue working. Build logs in ClickHouse are unaffected.

**Recovery**: Docker's `restart: unless-stopped` policy auto-restarts the container. Health check verifies recovery.

## Kafka Down

**What happens**: Build logs stop streaming. Build server's `publishLog()` calls fail. Builds may hang or crash.

**Impact**: Deployments start but logs don't appear in real-time. If all 3 brokers are down, builds fail because the producer can't connect.

**Recovery**: With 3-node replication, losing 1 broker is tolerable (quorum = 2/3). Full cluster failure requires manual restart.

## Docker Down

**What happens**: All containers stop. The entire application is offline.

**Impact**: Complete service outage. No frontend, no backend, no databases.

**Recovery**: Docker systemd service auto-restarts. User data bootstrap enabled Docker with `systemctl enable docker`.

## Nginx Down

**What happens**: Port 80 stops responding. External requests fail.

**Impact**: Backend is still running on port 5005, but clients hitting port 80 get connection refused.

**Recovery**: `restart: unless-stopped` auto-restarts. Direct backend access on port 5005 available as fallback.

## S3 Down

**What happens**: Deployed websites become inaccessible. New file uploads fail.

**Impact**: All live deployments show errors. New deployments fail at upload step.

**Recovery**: S3 has 99.99% availability (design: 99.999999999% durability). AWS resolves S3 outages quickly.

## EC2 Down

**What happens**: Everything goes offline. It's a single point of failure.

**Impact**: Complete outage until instance is restored.

**Recovery**: Elastic IP persists. Restart the instance, containers auto-start with `unless-stopped`.

## Docker Image Missing

**What happens**: `vortex-build-server:latest` image doesn't exist. Deploy fails.

**Impact**: No deployments can be triggered.

**Recovery**: `deploy.controller.js` auto-builds the image: `docker build -t vortex-build-server:latest /home/ubuntu/vortex/build-server`.

## Health Check Failure

**What happens**: CD pipeline's 3-minute health check loop fails. Deployment is marked as failed.

**Impact**: GitHub Actions reports failure. Debug logs are collected from the EC2 host.

**Recovery**: The CD pipeline SSHs into EC2 and collects `docker ps`, `docker compose logs`, and individual container logs for diagnosis.

## Terraform State Lost

**What happens**: Terraform doesn't know what infrastructure exists. Running `apply` would create duplicates.

**Impact**: Can't modify or destroy infrastructure via Terraform.

**Recovery**: Use `terraform import` to re-associate existing resources with state. Or destroy manually in AWS Console and re-apply.

## IAM Credentials Expired

**What happens**: Build server can't upload to S3. Ansible can't inject valid credentials.

**Impact**: Builds complete but uploads fail. "Deployment failed due to upload errors."

**Recovery**: Rotate credentials in GitHub Secrets and Ansible variables. Re-deploy.

## Disk Full

**What happens**: Docker can't write container layers. MongoDB/ClickHouse can't write data.

**Impact**: All writes fail. Services crash.

**Recovery**: `docker system prune` to remove unused images/containers. Increase EBS volume size.

## Memory Full (OOM)

**What happens**: Linux OOM Killer selects and kills a process.

**Impact**: Usually kills the process using the most memory (often a build container or ClickHouse).

**Recovery**: 4GB swap space provides buffer. `restart: unless-stopped` restarts killed containers.

## CPU 100%

**What happens**: System becomes unresponsive. API latency spikes.

**Impact**: Requests time out. Health checks fail.

**Recovery**: Build containers are the most CPU-intensive (`npm install` + `npm run build`). Limit concurrent builds. CloudWatch alerts for CPU > 80%.

## Rollback Strategy

Vortex has a `scripts/rollback.sh`:

```bash
git reset --hard HEAD~1
docker compose down --remove-orphans
docker compose up -d --build
```

This rolls back to the previous Git commit and rebuilds all containers.

---
---

# SECTION 22: SCALABILITY

## Current Architecture Limits

| Component | Current Capacity | Bottleneck |
|-----------|-----------------|-----------|
| **EC2 (t3.large)** | 2 vCPU, 8GB RAM | ~5 concurrent builds |
| **Kafka (3-node)** | ~100K msg/sec | Way beyond current needs |
| **MongoDB** | ~10K ops/sec | Way beyond current needs |
| **ClickHouse** | ~1M inserts/sec | Way beyond current needs |
| **Docker** | ~20 containers | Disk and memory limited |

## Horizontal Scaling Path

### Phase 1: Vertical Scaling
- Upgrade EC2 to `m5.xlarge` (4 vCPU, 16GB RAM)
- Supports ~20 concurrent builds

### Phase 2: Service Separation
- Move MongoDB to **MongoDB Atlas** (managed)
- Move Kafka to **Amazon MSK** (managed)
- Move ClickHouse to **ClickHouse Cloud** (managed)
- EC2 only runs application + Nginx + build containers

### Phase 3: Container Orchestration
- Migrate to **AWS ECS Fargate** for build containers
- Each build runs as a Fargate task (serverless containers)
- Supports 100+ concurrent builds

### Phase 4: Full Cloud-Native
- **EKS** (Kubernetes) for application orchestration
- **ALB** (Application Load Balancer) for traffic distribution
- **CloudFront** CDN for global S3 asset delivery
- **Auto Scaling Group** for EC2 instances

---
---

# SECTION 23: TRADEOFFS

## Technology Tradeoffs Summary

| Decision | Chose | Over | Why |
|----------|-------|------|-----|
| **React** | React 19 | Angular, Vue | Largest ecosystem, most job postings, Vite integration |
| **Express** | Express 5 | FastAPI, NestJS | Same language as frontend, minimal overhead, full control |
| **MongoDB** | MongoDB 6 | PostgreSQL | Document model fits users/deployments, no joins needed |
| **Docker** | Docker CE | VMs, bare metal | Isolation, reproducibility, industry standard |
| **EC2** | EC2 | ECS, Kubernetes | Full control, simpler for single-instance, Docker native |
| **Terraform** | Terraform | CloudFormation | Multi-cloud, HCL readable, massive community |
| **Kafka** | Kafka 3.8 | RabbitMQ, Redis Streams | Log persistence, ClickHouse native consumer, replay |
| **ClickHouse** | ClickHouse 23 | PostgreSQL, Elasticsearch | Columnar compression, sub-second analytics, Kafka Engine |
| **GitHub Actions** | GH Actions | Jenkins, GitLab CI | Zero setup, native GitHub integration, free tier |
| **Ansible** | Ansible | Bash scripts | Idempotent, templates, modules, readable YAML |
| **Nginx** | Nginx 1.25 | HAProxy, Traefik | Most documented, lightweight Alpine image, proven |
| **Vite** | Vite 6 | CRA, Next.js | Instant HMR, fast builds, modern ES modules |
| **TailwindCSS** | Tailwind 4 | Vanilla CSS, SCSS | Utility-first, no CSS files to manage, rapid prototyping |
| **Redux Toolkit** | RTK | Context API, Zustand | Industry standard, persist integration, mature devtools |

---
---

# SECTION 24: WHY THIS TECHNOLOGY

## Why React? vs Angular vs Vue

**React**: Component-based, virtual DOM, hooks system. The largest ecosystem of libraries, tools, and tutorials. Used by Meta, Netflix, Airbnb, and most startups.

**Angular**: Full framework with TypeScript, RxJS, dependency injection. Better for enterprise apps with complex forms and strict architecture requirements. Steeper learning curve.

**Vue**: Progressive framework, easier to learn. Great documentation. Smaller ecosystem and job market compared to React.

**Vortex chose React because**: JavaScript full-stack consistency, Vite's superior DX, and the widest job market relevance.

## Why Express? vs FastAPI vs NestJS

**Express**: Minimal, unopinionated. You build the architecture. Perfect for APIs with <20 routes.

**FastAPI**: Python. Faster for async I/O, built-in type validation. But requires a different language from the frontend.

**NestJS**: TypeScript, decorators, dependency injection. Great for large enterprise backends with 100+ routes. Overkill for Vortex's 5 route files.

## Why MongoDB? vs PostgreSQL

**MongoDB**: Flexible documents, no migrations, Mongoose ODM. Perfect for user profiles and deployment metadata.

**PostgreSQL**: ACID transactions, complex joins, strong consistency. Better for financial data or highly relational data models.

## Why Docker? vs VM

**Docker**: Millisecond startup, 10-500MB images, process-level isolation. Perfect for microservices and CI/CD.

**VM**: Full OS isolation, hypervisor-based. 1-10GB images, minute-long startup. Better for running untrusted workloads needing kernel-level isolation.

## Why EC2? vs ECS vs Kubernetes

**EC2**: Full control, SSH access, any software. Simple for single-instance deployments.

**ECS**: AWS container orchestration. Better for multi-container, multi-instance deployments. Adds complexity.

**Kubernetes**: Industry standard for container orchestration. Supports thousands of containers across hundreds of nodes. Massive complexity for a single-developer project.

## Why Terraform? vs CloudFormation

**Terraform**: Multi-cloud, clean HCL syntax, massive community, explicit state management.

**CloudFormation**: AWS-only, verbose JSON/YAML, AWS-managed state. Lock-in to AWS.

## Why Kafka? vs RabbitMQ vs Redis Streams

**Kafka**: Persistent log, replay capability, built-in ClickHouse consumer. Best for event streaming.

**RabbitMQ**: Message deleted after consumption, better for task queues. No persistence.

**Redis Streams**: In-memory, no native ClickHouse integration. Better for lightweight pub/sub.

## Why ClickHouse? vs PostgreSQL vs Elasticsearch

**ClickHouse**: Columnar compression, sub-second analytics on millions of rows, native Kafka Engine.

**PostgreSQL**: General-purpose RDBMS. Adequate for small log volumes, but 10-100x slower on analytical queries.

**Elasticsearch**: Full-text search engine. Better for unstructured text search. Higher resource requirements (4GB+ JVM heap).

## Why GitHub Actions? vs Jenkins vs GitLab vs CircleCI

**GitHub Actions**: Zero infrastructure, native GitHub integration, free 2000 min/month.

**Jenkins**: Self-hosted, Groovy pipelines. Maximum flexibility but requires server maintenance.

**GitLab CI**: Excellent but requires GitLab. Not ideal when repo is on GitHub.

**CircleCI**: Good cloud CI but adds another service to manage.

---
---

# SECTION 25: INTERVIEW QUESTIONS

## Architecture & Design

1. **Walk me through the Vortex architecture.**
2. **How does a deployment work end-to-end?**
3. **Why did you choose a microservices architecture?**
4. **How do services communicate with each other?**
5. **What happens when a user deploys a website?**
6. **How are build logs streamed in real-time?**
7. **Why Kafka → ClickHouse instead of directly writing to a database?**
8. **How does the build server isolation work?**

## Docker & Containerization

9. **What is Docker-in-Docker and how does Vortex use it?**
10. **Explain the Docker socket mounting approach.**
11. **What is the difference between named volumes and bind mounts?**
12. **How does Docker networking work in your Compose setup?**
13. **What happens if the Docker daemon crashes?**
14. **Explain container lifecycle management in Vortex.**

## Infrastructure & Cloud

15. **Walk me through your Terraform configuration.**
16. **What is the Terraform state file and why is it critical?**
17. **Explain your VPC architecture.**
18. **How do security groups protect your infrastructure?**
19. **What is user_data and when does it run?**
20. **Why Elastic IP instead of a regular public IP?**

## CI/CD & DevOps

21. **Explain your CI/CD pipeline.**
22. **What security checks run in CI?**
23. **How does the CD pipeline deploy to EC2?**
24. **What happens if the health check fails after deployment?**
25. **How does Ansible differ from running bash scripts?**

## Databases

26. **Why two databases (MongoDB + ClickHouse)?**
27. **When would you choose PostgreSQL over MongoDB?**
28. **How does ClickHouse consume from Kafka?**
29. **What is a Materialized View in ClickHouse?**
30. **How does the MergeTree engine work?**

## Security

31. **What security measures are implemented?**
32. **How do you handle authentication?**
33. **What is Helmet and what headers does it set?**
34. **How do you prevent SQL/NoSQL injection?**
35. **What is IMDSv2 and why is it enabled?**

## Scaling & Production

36. **How would you scale Vortex to handle 1000 concurrent deployments?**
37. **What is the single point of failure in your architecture?**
38. **How would you add zero-downtime deployments?**
39. **What monitoring would you add in production?**
40. **How would you implement auto-scaling?**

---
---

# SECTION 26: COUNTER QUESTIONS

Counter questions are what you ask the interviewer when they challenge your technology choices. They demonstrate depth of understanding.

## When They Say "Why Not X?"

### "Why not use Kubernetes?"

> "Kubernetes is excellent for orchestrating hundreds of containers across multiple nodes. But Vortex runs on a single EC2 instance with 8 long-running services and ephemeral build containers. Docker Compose handles this perfectly with zero operational overhead. Adding Kubernetes would mean managing etcd, the control plane, kubelet, kube-proxy, and potentially a managed service like EKS that costs $73/month just for the control plane — all for a workload that fits on one machine. I'd introduce Kubernetes when we need multi-node scaling, auto-healing across instances, or rolling updates across a fleet."

### "Why not use AWS Lambda for builds?"

> "Lambda has a 15-minute execution limit and 10GB ephemeral storage. A `npm install` for a large project can take 5+ minutes, and `node_modules` can exceed several GB. Lambda also doesn't support Docker-in-Docker or long-running processes well. EC2 with Docker gives unlimited execution time, full filesystem access, and the ability to install any toolchain (Java, Maven, Gradle, Python)."

### "Why not use Vercel directly?"

> "The entire point of Vortex is to understand what happens underneath Vercel. Using Vercel would be like answering 'How does a car engine work?' with 'I use Uber.' Building Vortex taught me Docker containerization, Kafka streaming, Terraform IaC, Ansible automation, and production debugging — skills you don't learn from using a managed platform."

### "Why not WebSockets for log streaming?"

> "WebSockets would provide true real-time streaming (push model). Vortex currently uses polling — the frontend fetches logs every few seconds. WebSockets are better for real-time, but polling is simpler to implement, cache, and debug. In production, I'd upgrade to WebSockets or Server-Sent Events (SSE) for true push-based log streaming."

### "Why not use a managed database like MongoDB Atlas?"

> "For learning, running MongoDB in Docker teaches you about replica sets, connection strings, data persistence with volumes, and container health checks. Atlas abstracts all of this. In production, I'd absolutely use Atlas for automatic backups, scaling, and monitoring."

### "Why is your ClickHouse query vulnerable to SQL injection?"

> "Good catch. The current log query interpolates the deployment ID directly into the SQL string. In production, I'd use parameterized queries with ClickHouse's query parameters. However, the deployment ID is generated server-side (not user-input), which limits the attack surface."

---
---

# SECTION 27: SYSTEM DESIGN

## If Asked: "Design a Vercel-like Deployment Platform"

### Requirements

**Functional**:
- Users register, login, connect GitHub
- Select repo + branch → trigger deployment
- Build in isolated environment
- Stream build logs in real-time
- Upload build output to CDN/object storage
- Serve deployed website at a unique URL

**Non-Functional**:
- Handle 1000+ concurrent builds
- Build logs visible within 1 second of generation
- 99.9% uptime for deployed websites
- Sub-second page load for deployed sites

### High-Level Design

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Frontend   │────▶│   API Server  │────▶│  Build Queue    │
│   (React)    │     │   (Express)   │     │  (Kafka/SQS)    │
└─────────────┘     └──────┬───────┘     └────────┬────────┘
                           │                       │
                    ┌──────┴───────┐        ┌──────┴────────┐
                    │   Auth DB     │        │  Build Workers │
                    │  (MongoDB/    │        │  (ECS Fargate/ │
                    │   PostgreSQL) │        │   Docker)      │
                    └──────────────┘        └──────┬────────┘
                                                    │
                    ┌──────────────┐        ┌──────┴────────┐
                    │   Log Store   │◀───────│  Log Pipeline  │
                    │  (ClickHouse/ │        │  (Kafka)       │
                    │   TimescaleDB)│        └───────────────┘
                    └──────────────┘
                                            ┌───────────────┐
                                            │  Object Store  │
                                            │  (S3/GCS)      │
                                            └───────┬───────┘
                                                    │
                                            ┌───────┴───────┐
                                            │  CDN           │
                                            │  (CloudFront)  │
                                            └───────────────┘
```

### Key Design Decisions

1. **Build isolation**: Each build runs in an ephemeral container (ECS Fargate task or Docker container). This prevents interference between builds and provides security isolation.

2. **Log pipeline**: Kafka decouples log producers (build workers) from consumers (log storage). This handles burst traffic and allows multiple consumers (dashboard, alerting, analytics).

3. **Storage**: S3 for build artifacts + CloudFront CDN for global edge delivery.

4. **Database split**: Transactional data (users, deployments) in MongoDB/PostgreSQL. Time-series data (logs) in ClickHouse.

5. **Queue-based builds**: A build queue (Kafka/SQS) ensures builds are processed in order, with backpressure handling when workers are saturated.

### Capacity Estimation

For 1000 concurrent builds:
- Each build: ~500MB RAM, 1 vCPU, 5-10 minutes
- Build workers needed: ~1000 (if all concurrent)
- Kafka throughput: ~50K logs/sec (50 logs/sec per build × 1000 builds)
- ClickHouse: ~50K inserts/sec (well within capacity)
- S3: ~1000 uploads/5min = ~3 uploads/sec (trivial)

---
---

# SECTION 28: PRODUCTION IMPROVEMENTS

## What Would Change for Production

| Area | Current | Production |
|------|---------|-----------|
| **Database** | MongoDB in Docker | MongoDB Atlas (managed, backups, scaling) |
| **Kafka** | 3-node Docker | Amazon MSK (managed Kafka) |
| **ClickHouse** | Single node Docker | ClickHouse Cloud (managed, replicated) |
| **Build Workers** | Docker on single EC2 | ECS Fargate (serverless containers) |
| **Load Balancing** | Single Nginx | AWS ALB + Auto Scaling Group |
| **CDN** | Direct S3 URLs | CloudFront with edge caching |
| **SSL** | HTTP only | Let's Encrypt + Nginx SSL termination |
| **Domain** | Elastic IP | Custom domain with Route 53 |
| **Monitoring** | Health endpoints only | Prometheus + Grafana + CloudWatch |
| **Logging** | Docker stdout | Centralized logging (ELK or CloudWatch Logs) |
| **State** | Local terraform.tfstate | S3 backend + DynamoDB locking |
| **Secrets** | GitHub Secrets | AWS Secrets Manager or HashiCorp Vault |
| **Auth** | JWT only | OAuth 2.0 (GitHub OAuth for login) |
| **Rate Limiting** | Express middleware | AWS WAF (Web Application Firewall) |
| **Backup** | Manual script | Automated MongoDB Atlas snapshots |
| **Multi-region** | Single AZ | Multi-AZ with cross-region replication |
| **Caching** | None | Redis for session cache + API response cache |
| **WebSocket** | Polling for logs | WebSocket or SSE for real-time log streaming |
| **Blue-Green** | None | Blue-green deployment with ALB target groups |
| **Canary** | None | Canary releases with weighted ALB routing |

---
---

# SECTION 29: DEBUGGING STORIES

These are real problems faced during development and deployment of Vortex. Each one demonstrates genuine engineering experience.

## Story 1: GitHub Secret Scanning Blocked Push

**The Problem**: Attempted to push code to GitHub. Push was rejected with `GH013: Repository rule violations found`.

**The Reason**: The `.env` file or code contained AWS access keys. GitHub's push protection automatically scans for high-entropy strings that look like secrets and blocks the push to prevent credential leaks.

**The Fix**: 
1. Removed the exposed secret from the code.
2. Added `.env` to `.gitignore`.
3. Used `git filter-branch` or `git rebase -i` to remove the secret from Git history.
4. Rotated the AWS credentials (old ones are compromised once pushed, even briefly).

**What I Learned**: Never commit secrets to Git. Use environment variables, GitHub Secrets, or AWS Secrets Manager. Even if you delete the file, the secret exists in Git history.

**Commit evidence**: `96b2d4b Remove exposed secret`

## Story 2: vCPU Limit Exceeded

**The Problem**: `terraform apply` failed with `VcpuLimitExceeded` error when launching EC2 instance.

**The Reason**: AWS has per-region vCPU quotas for on-demand instances. The default quota for new accounts in `eu-north-1` was insufficient for the requested instance type.

**The Fix**:
1. First tried smaller instance types: `t3.large` → `t3.nano` → `t3.micro` → `t2.micro` → `t4g.micro` (ARM).
2. When all failed in `eu-north-1`, switched region to `us-east-1` which has higher default quotas.
3. Updated all Terraform variables and Ansible configuration for the new region.

**What I Learned**: Always check AWS service quotas before provisioning. New accounts have low vCPU limits. You can request increases via AWS Support (takes 24-48 hours) or use regions with higher defaults.

**Commit evidence**: `2ed6d1a config: update region to us-east-1 to bypass regional vCPU quota constraints`, `65a6b45 config: update instance_type to free tier t4g.micro`, `eba4a89 config: update instance_type to t3.nano`, `d26eba2 config: update instance_type to t2.micro`, `2b4c36e config: update instance_type to t3.micro`

## Story 3: SSH Authentication Failures in CD Pipeline

**The Problem**: The CD pipeline couldn't SSH into the EC2 instance. SSH connection timed out or returned authentication errors.

**The Reason**: Multiple issues:
1. The SSH key format didn't match what AWS expected.
2. The username wasn't explicitly set (defaulted to root instead of ubuntu).
3. The private key file permissions were too open.
4. The EC2 host key wasn't in `known_hosts`, causing interactive verification prompts that locked up the runner.

**The Fix**:
1. Generated a 4096-bit RSA key pair compatible with AWS API validation.
2. Explicitly set `ansible_user=ubuntu` in the CD workflow.
3. Used `chmod 600` for the private key.
4. Added `ssh-keyscan -H $EC2_HOST >> ~/.ssh/known_hosts` to pre-register the host key.

**What I Learned**: SSH key management is critical. Always verify key format, username, permissions (600 for keys), and pre-register host keys in automated pipelines.

**Commit evidence**: `89042c7 fix: set explicit username ubuntu in CD`, `96db20a fix: generate 4096-bit RSA SSH key pair`, `a45689f fix: register native AWS key pair`, `da9246f fix: authorize deployment SSH key`

## Story 4: Frontend API Pointing to localhost

**The Problem**: After deploying the frontend to Vercel, all API calls failed. The frontend was trying to reach `http://localhost:5000` instead of the production backend.

**The Reason**: The frontend's API base URL was hardcoded to `localhost:5000` for development. When deployed to Vercel, `localhost` on the user's browser doesn't have the Express server.

**The Fix**:
1. First tried setting an explicit global axios base URL to the live EC2 IP: `http://52.45.15.38:5005`.
2. This caused **mixed content blocking** — Vercel serves over HTTPS, but the EC2 backend was HTTP-only. Browsers block HTTP requests from HTTPS pages.
3. Solved by using **Vercel Edge Rewrites** — configured `vercel.json` to proxy `/api/*` through Vercel's edge to the EC2 backend, maintaining HTTPS end-to-end.

**What I Learned**: CORS and mixed content are real production issues. Never hardcode `localhost` in frontend code. Use environment variables or edge proxies to route API traffic.

**Commit evidence**: `a917ace fix: set explicit global axios baseURL`, `f9eac87 fix: proxy /api/* via Vercel edge rewrite`, `40173b7 fix: route Vercel proxy to production Nginx port 80`, `aa998d7 fix: route api queries relatively through vercel https rewrite proxy`

## Story 5: Nginx 502 Bad Gateway

**The Problem**: After deployment, accessing the EC2 IP returned `502 Bad Gateway`.

**The Reason**: Nginx started before the backend container was fully healthy. The `depends_on: service_healthy` condition wasn't catching all startup timing issues, especially DNS cache staleness.

**The Fix**:
1. Improved health check configuration with proper `retries` and `delay` intervals.
2. Changed the Ansible handler from `docker compose restart` to `docker compose down && docker compose up -d` to resolve stale DNS cache issues.
3. Added the `flush_handlers` meta task before health checks.

**What I Learned**: Container orchestration timing is tricky. Health checks must be robust. Docker's internal DNS can cache stale IPs when containers are recreated. `down + up` is safer than `restart` for resetting DNS state.

**Commit evidence**: `f899692 fix: use docker compose down/up handler to resolve stale DNS cache`, `889361f Improve deployment health checks and CD diagnostics`

## Story 6: Build Server Image Not Found

**The Problem**: Deployments failed with "Image not found" for `vortex-build-server:latest`.

**The Reason**: The build server Docker image wasn't being built automatically. After a fresh deployment or Docker image prune, the image didn't exist.

**The Fix**:
1. Added auto-build logic in `deploy.controller.js`: if `docker image inspect` fails, automatically run `docker build`.
2. Added a build step in the Ansible deploy tasks: `docker build -t vortex-build-server:latest {{ project_directory }}/build-server`.
3. Added health verification in Ansible: `docker image inspect vortex-build-server:latest` to ensure the image exists.

**What I Learned**: Always have self-healing mechanisms. Don't assume Docker images exist — check and build on-demand.

**Commit evidence**: `8bfdb08 fix: automate build-server compilation and inject AWS S3 credentials`

## Story 7: Ansible Template Not Staged

**The Problem**: CD pipeline failed because the Ansible template file (`.env.j2`) wasn't committed to Git.

**The Reason**: The template was created locally but never staged and committed. Git doesn't track untracked files.

**The Fix**: `git add` the template file and CI workflow, then commit and push.

**What I Learned**: Always verify new files are staged before pushing. Use `git status` religiously.

**Commit evidence**: `7a29b78 fix: stage missing env.j2 template and CI workflow`

## Story 8: CD SSH Command Timeout

**The Problem**: CD pipeline timed out during the Ansible deployment step.

**The Reason**: The default SSH action timeout was too short. Docker Compose pulling images and building containers for 8 services takes significant time, especially on a fresh EC2 instance without cached images.

**The Fix**: Increased the SSH command timeout from the default to 30 minutes to accommodate heavy microservice provisioning.

**What I Learned**: First-time deployments are much slower than subsequent ones due to Docker image pulls and npm installs. Always set generous timeouts for CI/CD steps that involve heavy operations.

**Commit evidence**: `98d2ebd fix: increase SSH action command timeout to 30m for heavy microservice provisioning`

---
---

# SECTION 30: REAL PROBLEMS FACED

This section consolidates every real engineering problem encountered during Vortex development, providing the structured format interviewers love.

---

### Problem 1: GitHub Push Protection Blocked Commit

```
Symptom     → git push rejected with GH013
Root Cause  → AWS credentials in committed .env file
Fix         → Remove secret, purge git history, rotate credentials
Learning    → Never commit secrets. Use .gitignore + env vars.
```

---

### Problem 2: AWS vCPU Quota Exceeded

```
Symptom     → terraform apply → VcpuLimitExceeded
Root Cause  → New AWS account has low default vCPU quota per region
Fix         → Tried 5 instance types, 2 architectures, finally switched regions
Learning    → Check AWS service quotas BEFORE provisioning. Use us-east-1 for generous defaults.
```

---

### Problem 3: SSH Key Authentication Failure

```
Symptom     → CD pipeline SSH connection timeout / auth failure
Root Cause  → Wrong key format, missing username, loose file permissions
Fix         → 4096-bit RSA key, explicit ubuntu user, chmod 600, ssh-keyscan
Learning    → SSH is sensitive to key format, permissions, and host key verification.
```

---

### Problem 4: Mixed Content Blocking (HTTPS → HTTP)

```
Symptom     → Frontend API calls fail silently after Vercel deployment
Root Cause  → Vercel serves HTTPS, backend is HTTP. Browsers block mixed content.
Fix         → Vercel Edge Rewrites to proxy /api/* maintaining HTTPS
Learning    → Always use HTTPS in production. Edge proxies solve mixed content.
```

---

### Problem 5: Nginx 502 Bad Gateway

```
Symptom     → http://EC2_IP returns 502
Root Cause  → Backend container not ready when Nginx starts. DNS cache stale.
Fix         → Health check conditions, docker compose down+up instead of restart
Learning    → Container startup ordering needs health checks, not just depends_on.
```

---

### Problem 6: Docker Build Server Image Missing

```
Symptom     → Deployment fails: "Image vortex-build-server:latest not found"
Root Cause  → Image not pre-built after fresh deployment / docker prune
Fix         → Auto-build in controller + Ansible build step + health verification
Learning    → Never assume. Always verify and self-heal.
```

---

### Problem 7: Ansible Template File Not Committed

```
Symptom     → Ansible playbook fails: "template not found"
Root Cause  → .env.j2 template created but never git-added
Fix         → git add + commit the template file
Learning    → Always check git status. New files aren't tracked automatically.
```

---

### Problem 8: CD Pipeline Timeout During First Deploy

```
Symptom     → SSH step times out after 10 minutes
Root Cause  → First deploy pulls all Docker images + npm install = 20+ minutes
Fix         → Increased SSH timeout to 30 minutes
Learning    → Cold starts are expensive. Set generous timeouts for CI/CD.
```

---

### Problem 9: Vercel Proxy Routing to Wrong Port

```
Symptom     → API calls from Vercel frontend return 502/connection refused
Root Cause  → Vercel rewrite pointed to wrong port (5000 instead of 80/5005)
Fix         → Updated vercel.json rewrite destination to Nginx port 80
Learning    → Docker port mapping (5005→5000) means external access uses 5005.
```

---

### Problem 10: Environment Variables Not Available in Docker

```
Symptom     → Backend crashes on startup: "MONGO_URI undefined"
Root Cause  → .env file not generated on EC2 before docker compose up
Fix         → Ansible template generates .env first, then docker compose up
Learning    → Configuration must be generated BEFORE services start.
```

---

### Problem 11: Docker Compose DNS Cache Stale

```
Symptom     → After docker compose restart, services can't resolve each other
Root Cause  → Docker's embedded DNS caches IPs. Restart reuses stale cache.
Fix         → Use docker compose down + up instead of restart (resets DNS)
Learning    → Docker DNS caching is a real issue. Down+up is safer than restart.
```

---

### Problem 12: Multiple Instance Type Failures

```
Symptom     → terraform apply fails for t3.large, t3.micro, t2.micro, t4g.micro
Root Cause  → Each instance type has its own vCPU quota per region
Fix         → Systematically tried smaller types, then switched to us-east-1
Learning    → AWS quotas are per-instance-family AND per-region. Know the limits.
```

---

> **Final Note**: These debugging stories are not hypothetical. Every one is evidenced by Git commit messages in the Vortex repository. They demonstrate real production engineering experience — not textbook knowledge, but battle-tested problem-solving skills.

---
---

# APPENDIX: QUICK REFERENCE

## Commands Cheat Sheet

```bash
# Development
npm run start-frontend         # Start Vite dev server (port 3000)
npm run start-backend          # Start Express server (port 5000)

# Docker
docker compose -f services/docker-compose.yml up -d --build   # Start all services
docker compose -f services/docker-compose.yml down             # Stop all services
docker compose -f services/docker-compose.yml logs -f          # Follow logs
docker build -t vortex-build-server:latest build-server/       # Build build-server image

# Terraform
cd infra/terraform
terraform init                 # Initialize providers
terraform plan                 # Preview changes
terraform apply                # Apply changes
terraform destroy              # Destroy everything

# Ansible
ansible-playbook -i infra/ansible/inventory.ini infra/ansible/playbook.yml

# Operations
bash scripts/deploy.sh         # Deploy latest code
bash scripts/rollback.sh       # Rollback to previous commit
bash scripts/health-check.sh   # Check all services health
bash scripts/backup.sh         # Backup MongoDB + ClickHouse

# Kafka Benchmark
bash benchmark-kafka.sh        # Run producer/consumer performance test
```

## Environment Variables

| Variable | Used By | Purpose |
|----------|---------|---------|
| `MONGO_URI` | Backend | MongoDB connection string |
| `CLICKHOUSE_URL` | Backend | ClickHouse HTTP endpoint |
| `KAFKA_BROKER` | Backend, Build Server | Kafka bootstrap server |
| `JWT_SECRET` | Backend | JWT signing key |
| `GITHUB_TOKEN` | Backend | GitHub API authentication |
| `AWS_ACCESS_KEY_ID` | Build Server | S3 upload authentication |
| `AWS_SECRET_ACCESS_KEY` | Build Server | S3 upload authentication |
| `S3_BUCKET` | Build Server | Target S3 bucket for uploads |
| `VITE_API_URL` | Frontend | Backend API base URL |
| `VITE_API_TARGET` | Frontend (Vite) | Proxy target for development |
| `CORS_ORIGIN` | Backend | Allowed CORS origins |

## Port Mapping

| Internal Port | External Port | Service |
|--------------|--------------|---------|
| 3000 | 3005 | Frontend (Vite) |
| 5000 | 5005 | Backend (Express) |
| 80 | 80 | Nginx |
| 19092 | 19092 | Kafka Broker 1 |
| 19092 | 29092 | Kafka Broker 2 |
| 19092 | 39092 | Kafka Broker 3 |
| 8080 | 8080 | Redpanda Console |
| 8123 | 8123 | ClickHouse HTTP |
| 9000 | 9000 | ClickHouse Native |
| 27017 | 27017 | MongoDB |

---

> **End of Document**
> 
> This document contains everything you need to explain, defend, and deeply discuss every aspect of the Vortex project in any technical interview. Study the WHY behind every decision, practice the debugging stories, and prepare counter-arguments for alternative technology choices.
