# 🌀 VORTEX — Complete Technical Interview Handbook

> **Simulated Senior Technical Interview · 300+ Questions · Grounded in Your Actual Codebase**
> Prepared for a Principal Engineer / Staff Backend / DevOps Architect level interview

---

# TABLE OF CONTENTS

1. [Section 1 — Project Introduction](#section-1--project-introduction)
2. [Section 2 — Architecture](#section-2--architecture)
3. [Section 3 — Frontend (React / Vite / Redux)](#section-3--frontend)
4. [Section 4 — Backend (Express / Node.js)](#section-4--backend)
5. [Section 5 — Database (MongoDB)](#section-5--database-mongodb)
6. [Section 6 — Apache Kafka](#section-6--apache-kafka)
7. [Section 7 — ClickHouse](#section-7--clickhouse)
8. [Section 8 — Docker](#section-8--docker)
9. [Section 9 — Terraform](#section-9--terraform)
10. [Section 10 — Ansible](#section-10--ansible)
11. [Section 11 — GitHub Actions CI/CD](#section-11--github-actions-cicd)
12. [Section 12 — AWS Infrastructure](#section-12--aws-infrastructure)
13. [Section 13 — Nginx](#section-13--nginx)
14. [Section 14 — Security](#section-14--security)
15. [Section 15 — DevOps Practices](#section-15--devops-practices)
16. [Section 16 — Low Level Design](#section-16--low-level-design)
17. [Section 17 — System Design / Scaling](#section-17--system-design--scaling)
18. [Section 18 — Failure Scenarios](#section-18--failure-scenarios)
19. [Section 19 — Tradeoffs](#section-19--tradeoffs)
20. [Section 20 — Resume Deep Dive](#section-20--resume-deep-dive)
21. [Section 21 — Behavioral](#section-21--behavioral)
22. [Section 22 — Coding Challenges](#section-22--coding-challenges)
23. [Section 23 — Rapid Fire (100 Questions)](#section-23--rapid-fire)
24. [Section 24 — Counter-Question Drill Trees](#section-24--counter-question-drill-trees)

---

# SECTION 1 — PROJECT INTRODUCTION

---

## Q1. "Tell me about Vortex."

**Why interviewer asks:** First impression. They want a crisp elevator pitch, not a rambling walkthrough. They are testing whether you can articulate complex systems simply.

### Ideal Answer
"Vortex is a cloud-native static website deployment platform I built from scratch — essentially an open-source Vercel alternative. Users register, connect their GitHub profile, select a repository and branch, and Vortex clones it, detects the build strategy (React/Vite/Next.js/static/Java), runs the build inside an isolated Docker container, streams real-time build logs through a 3-node Kafka cluster into ClickHouse for persistent storage, uploads the compiled assets to AWS S3, and serves the deployed site via an S3 URL. The entire infrastructure — VPC, EC2, security groups, IAM roles — is provisioned with Terraform, configured with Ansible, and deployed through a GitHub Actions CI/CD pipeline."

### Simple Explanation
It's a website that deploys other websites. Like Vercel, but I built it myself to learn cloud-native architecture end-to-end.

### Deep Explanation
Vortex is a full-stack platform with 4 independently deployable components:
1. **React Frontend** (Vite, Redux Toolkit with persistence, Axios, React Router)
2. **Express.js Backend** (JWT auth, REST APIs, GitHub integration via Octokit, ClickHouse log queries, Docker container orchestration)
3. **Build Server** (Ubuntu-based Docker container that clones repos, detects build strategies, runs builds, streams logs to Kafka, uploads artifacts to S3)
4. **Infrastructure Layer** (3-node KRaft Kafka cluster, ClickHouse with Kafka Engine + materialized views, MongoDB, Nginx reverse proxy, all orchestrated via Docker Compose on a Terraform-provisioned AWS EC2 t3.large instance)

### Architecture Explanation
```
User → React Frontend → Nginx Reverse Proxy → Express Backend
                                                      ↓
                                          Docker Run (Build Server Container)
                                                      ↓
                                          Git Clone → Detect Strategy → npm install/build
                                                      ↓
                                          Kafka Producer (build-logs topic)
                                                      ↓
                               ┌─────────────────────────────────────┐
                               │  3-Node KRaft Kafka Cluster         │
                               │  (kafka-1, kafka-2, kafka-3)        │
                               └─────────────────────────────────────┘
                                                      ↓
                                          ClickHouse Kafka Engine Table
                                                      ↓
                                          Materialized View → MergeTree Table
                                                      ↓
                                          Frontend polls /api/logs/:id
                                                      ↓
                                          S3 Upload (batched, 5 files at a time)
```

### Real Production Example
"When a user deploys a React/Vite project, the backend spawns a Docker container from the `vortex-build-server:latest` image. The container runs `main.sh`, which clones the repo, checks out the branch, injects user-supplied env vars as a `.env` file parsed from JSON using `jq`, then runs `script.js`. The script detects `vite.config.js` and runs `npm install && npm run build`. Every stdout/stderr line is published to Kafka topic `build-logs`. ClickHouse ingests via a Kafka Engine table and a materialized view writes to a MergeTree table ordered by `created_at`. The frontend polls the backend, which queries ClickHouse with `SELECT ... WHERE deployment_id = '...' ORDER BY created_at ASC`. After build completes, the `dist/` folder is uploaded to S3 in batches of 5 concurrent files."

### Common Mistakes
- ❌ Saying "it's like Vercel" without explaining HOW it works
- ❌ Not mentioning the Kafka log pipeline (this is the most impressive part)
- ❌ Forgetting to mention infrastructure-as-code (Terraform + Ansible)
- ❌ Making it sound like a tutorial project instead of an engineered system
- ❌ Not mentioning the Docker-in-Docker pattern for build isolation

### What Interviewer Expects
| Level | What they want to hear |
|-------|----------------------|
| Junior | Basic description of what it does |
| Mid | Architecture components and how they connect |
| Senior | Why each technology was chosen, tradeoffs considered |
| Staff/Principal | Production concerns: fault tolerance, scaling bottlenecks, security gaps |

### Counter Questions
1. "Why did you build this instead of contributing to an existing open-source deployment tool?"
2. "What's the most complex bug you encountered?"
3. "How long did this take to build?"
4. "What would you do differently if you started over?"
5. "How does this compare to Vercel's actual architecture?"

### Best Follow-up Answer
"If I rebuilt it, I'd replace the polling-based log fetching with Server-Sent Events or WebSockets for true real-time streaming, add a proper job queue (like BullMQ) for build orchestration instead of directly spawning Docker containers, implement proper multi-tenancy with resource quotas, and use Kubernetes instead of Docker Compose for the production orchestration layer."

### Things Never to Say
- "I followed a tutorial"
- "It's just a simple project"
- "I didn't understand Kafka, I just used it"

### Confidence Level Required: 🟢 HIGH — This is YOUR project.
### Difficulty: ⭐⭐ (Easy but sets the tone)
### Expected Duration: 3-5 minutes

---

## Q2. "What problem does Vortex solve?"

**Why interviewer asks:** They want to know if you understand the problem domain, not just the code.

### Ideal Answer
"Vortex solves the complexity of deploying static websites. Without a platform like this, a developer needs to manually: set up a build environment, install dependencies, run builds, handle build failures, configure S3 buckets with proper CORS and permissions, set up CDN distribution, manage deployments across environments, and monitor build logs. Vortex automates this entire pipeline. You point it at a GitHub repo, select a branch, and it handles everything — build strategy detection, isolated containerized builds, artifact storage, real-time log streaming, and deployment URL generation."

### Simple Explanation
Deploying websites is hard. Vortex makes it one-click.

### Deep Explanation
The deployment problem has several dimensions:
1. **Build Environment Isolation** — Different projects need different Node.js versions, Java JDKs, etc. Vortex solves this with Docker containers based on `ubuntu:focal` with Node 20, Java 11, and Maven pre-installed.
2. **Build Strategy Detection** — The build server's `detectBuildStrategy()` function checks for `package.json`, `pom.xml`, `build.gradle`, `vite.config.js`, and `next.config.js` to automatically determine the correct build command.
3. **Log Aggregation** — Build logs need to be captured, stored, and served in real-time. Vortex uses Kafka as a durable buffer and ClickHouse for queryable persistent storage.
4. **Artifact Management** — Built files need to be uploaded to a CDN-backed storage. Vortex uses S3 with batched parallel uploads.

### Counter Questions
1. "Why not just use a shell script that does `npm run build && aws s3 sync`?"
2. "Who are the target users? Individual developers or teams?"
3. "What types of projects can Vortex deploy? Only static sites or also server-rendered apps?"
4. "How does Vortex handle projects that need server-side rendering?"

### Best Follow-up Answer
"Currently Vortex only handles static site generation (SPAs, static HTML, compiled Java artifacts). It detects output directories — `dist/`, `build/`, or `target/` — and uploads those. For SSR frameworks like Next.js in full mode, you'd need a different architecture with persistent containers or serverless functions, which is a planned future enhancement."

### Confidence Level Required: 🟢 HIGH
### Difficulty: ⭐⭐
### Expected Duration: 2-3 minutes

---

## Q3. "Why did you build Vortex instead of using Vercel/Netlify?"

**Why interviewer asks:** They want to assess your motivation — are you a builder or just a consumer?

### Ideal Answer
"Three reasons. First, I wanted to deeply understand how platforms like Vercel work under the hood — the build pipeline, container orchestration, log streaming, and infrastructure automation. You can't learn that by just using Vercel. Second, I wanted hands-on experience with technologies I'd only read about — Kafka, ClickHouse, Terraform, Ansible — in a real, integrated system rather than isolated tutorials. Third, I wanted a portfolio project that demonstrates end-to-end systems thinking: from the React frontend through the API layer, into container orchestration, distributed log streaming, cloud infrastructure provisioning, and CI/CD automation."

### Counter Questions
1. "But Vercel is free for personal projects. Isn't this just reinventing the wheel?"
2. "What features does Vercel have that Vortex doesn't?"
3. "Would you actually use Vortex in production over Vercel?"
4. "What did you learn that you couldn't learn from documentation alone?"

### Common Mistakes
- ❌ Saying "Vercel is expensive" (it has a free tier)
- ❌ Not explaining what you learned
- ❌ Making it sound like you think Vortex is better than Vercel

### Confidence Level Required: 🟢 HIGH
### Difficulty: ⭐⭐
### Expected Duration: 2 minutes

---

## Q4. "What's the biggest technical challenge you faced?"

**Why interviewer asks:** This reveals your debugging skills, persistence, and depth of understanding.

### Ideal Answer
"The biggest challenge was the Docker-in-Docker pattern for build isolation. The main application container needs to spawn build-server containers on the host's Docker daemon. I solved this by mounting the Docker socket (`/var/run/docker.sock`) into the application container and installing `docker.io` inside it. But this introduced several issues:

1. **Network isolation**: The spawned build containers need to reach Kafka brokers. I solved this by connecting them to the `deployment` bridge network via `--network deployment` in the `docker run` command.
2. **Image availability**: The build-server image must exist on the host, not inside the application container. The deploy controller first does `docker image inspect` and if the image doesn't exist, it builds it from `/home/ubuntu/vortex/build-server`.
3. **Environment variable injection**: User-supplied env vars are passed as a JSON array through Docker `-e` flags, then parsed inside the container using `jq` into a `.env` file.
4. **Container cleanup**: If a container with the same deployment ID already exists, it's force-removed before spawning a new one."

### Deep Explanation
Looking at [deploy.controller.js](file:///Users/eshwarsai/Desktop/Vortex/backend/controllers/deploy.controller.js#L4-L63), the `deployProject` function:
- Validates required fields (repo, branch, username, deploymentId)
- Checks if the build-server Docker image exists via `execSync('docker image inspect ...')`
- Falls back to building the image if missing
- Removes any existing container with the same name
- Constructs a `docker run -d` command with 10+ environment variables
- Uses `docker wait` to asynchronously monitor container completion

### Counter Questions
1. "Mounting Docker socket is a security risk. How do you mitigate that?"
2. "What happens if two deployments use the same container name simultaneously?"
3. "Why not use Docker SDK for Node.js instead of shelling out to `docker` CLI?"
4. "What if the build container hangs indefinitely?"
5. "How do you limit resource consumption of build containers?"

### Best Follow-up Answer
"You're right about the security risk. Mounting the Docker socket gives the application container root-level access to the host. In a production system, I'd use a rootless Docker setup, or better yet, use a dedicated job queue (like BullMQ) with a separate build worker process that runs directly on the host rather than inside a container. For resource limits, I'd add `--memory` and `--cpus` flags to the `docker run` command, and implement a timeout with `docker stop` after a configurable duration."

### Confidence Level Required: 🟡 MEDIUM-HIGH (be honest about limitations)
### Difficulty: ⭐⭐⭐⭐
### Expected Duration: 5 minutes

---

## Q5. "What makes Vortex unique compared to other student/portfolio projects?"

### Ideal Answer
"Most portfolio projects are CRUD apps. Vortex integrates 15+ technologies into a cohesive production-grade system:
1. **Real-time log streaming** through a 3-node Kafka cluster (not a toy single-broker setup)
2. **ClickHouse integration** with Kafka Engine tables and materialized views for automatic log ingestion
3. **Infrastructure-as-code** with proper VPC, subnet, security group, IAM role Terraform modules
4. **Configuration management** with Ansible roles, Jinja2 templates, handlers, and health checks
5. **CI/CD** with separate CI (PR validation) and CD (push-to-main deployment) pipelines
6. **Security scanning** with Trivy, Gitleaks, and Snyk in the CI pipeline
7. **Build strategy auto-detection** supporting React, Vite, Next.js, Maven, and Gradle projects
8. **Operational scripts** for deployment, rollback, health checks, and backups"

### Confidence Level Required: 🟢 HIGH
### Difficulty: ⭐⭐
### Expected Duration: 2 minutes

---

## Q6. "Walk me through what happens when I click 'Deploy'."

**Why interviewer asks:** This is THE key question. It tests end-to-end understanding.

### Ideal Answer (Step by Step)

**Step 1 — Frontend**
The user on the Deploy page selects a repository and branch. The frontend generates a UUID for `deploymentId` and sends a POST to `/api/deploy/start` with `{ repo, branch, username, deploymentId, envVars }`.

**Step 2 — Backend receives request** ([deploy.controller.js](file:///Users/eshwarsai/Desktop/Vortex/backend/controllers/deploy.controller.js))
The `deployProject` controller validates required fields, then:
- Checks if the `vortex-build-server:latest` Docker image exists on the host
- If not, builds it from `/home/ubuntu/vortex/build-server`
- Removes any existing container with the same deployment ID
- Constructs and executes `docker run -d` with environment variables for REPO, BRANCH, USERNAME, DEPLOYMENT_ID, KAFKA_BROKER, S3 credentials, etc.
- Returns `200 OK` immediately (async deployment)
- Uses `docker wait` to monitor the container in the background

**Step 3 — Build container starts** ([main.sh](file:///Users/eshwarsai/Desktop/Vortex/build-server/main.sh))
- Clones the GitHub repository: `git clone https://github.com/{USERNAME}/{REPO}.git`
- Checks out the specified branch
- Parses user-supplied env vars from JSON using `jq` into a `.env` file
- Installs npm dependencies and runs `script.js`

**Step 4 — Build execution** ([script.js](file:///Users/eshwarsai/Desktop/Vortex/build-server/script.js))
- Connects the Kafka producer
- Calls `detectBuildStrategy()` — checks for `package.json`, `pom.xml`, `build.gradle`, `vite.config.js`, `next.config.js`
- Determines build command (e.g., `npm install && npm run build`)
- Executes build via `child_process.exec()`
- Every stdout/stderr line is published to Kafka topic `build-logs` via `publishLog()`

**Step 5 — Log ingestion pipeline** ([init-table.sql](file:///Users/eshwarsai/Desktop/Vortex/services/init-table.sql))
- ClickHouse has a `log_queue` table with ENGINE = Kafka that subscribes to `build-logs` topic
- A materialized view `kafka_queue` automatically transforms and inserts into `build_logs` MergeTree table
- Each log entry gets a UUID, timestamp, deployment_id, message, and level

**Step 6 — Frontend polls for logs**
- The frontend polls `GET /api/logs/{deploymentId}`
- The backend's [log.controller.js](file:///Users/eshwarsai/Desktop/Vortex/backend/controllers/log.controller.js) queries ClickHouse: `SELECT ... FROM build_logs WHERE deployment_id = '{id}' ORDER BY created_at ASC`
- Logs are displayed in real-time on the Deploy page

**Step 7 — S3 upload**
- After build completes, `script.js` detects the output directory (`dist/`, `build/`, or `target/`)
- Walks the directory tree, excluding `.git`, `node_modules`, `.env`
- Uploads files to S3 in batches of 5 concurrent files under `__outputs/{DEPLOYMENT_ID}/`
- Sets proper MIME types using the `mime-types` library

**Step 8 — Deployment record saved**
- Frontend calls `POST /api/deploy/create` with the deployment metadata and S3 URL
- Backend saves/updates the deployment record in MongoDB

### Counter Questions
1. "What happens if the build fails at Step 4?"
2. "How does the frontend know when the deployment is complete?"
3. "What if Kafka is down when the build server tries to publish logs?"
4. "Why polling instead of WebSockets for log streaming?"
5. "What's the maximum file size you can deploy?"
6. "How do you handle concurrent deployments?"
7. "What if the user's repo is private?"

### Confidence Level Required: 🟢 HIGH — You must know this cold.
### Difficulty: ⭐⭐⭐⭐⭐
### Expected Duration: 8-10 minutes

---

## Q7-Q10. Additional Introduction Questions

| # | Question | Key Points to Hit |
|---|----------|------------------|
| Q7 | "Who is the target user?" | Individual developers who want self-hosted deployment. Learning tool for understanding PaaS internals. |
| Q8 | "What's your biggest learning?" | End-to-end systems integration. Understanding that the hard part isn't individual services, it's making them work together reliably. |
| Q9 | "What would you improve?" | WebSocket log streaming, build queue with BullMQ, Kubernetes for orchestration, multi-tenancy, custom domains, SSL termination, build caching. |
| Q10 | "What's your contribution percentage?" | 100% — sole developer. Designed, built, deployed, and maintained everything. |

---

# SECTION 2 — ARCHITECTURE

---

## Q11. "Draw and explain the overall architecture of Vortex."

**Why interviewer asks:** Tests if you can visualize and communicate complex systems.

### Ideal Answer

```
┌──────────────────────────────────────────────────────────────────────┐
│                         AWS EC2 (t3.large)                          │
│                     VPC 10.0.0.0/16 · Subnet 10.0.1.0/24           │
│                         Elastic IP attached                         │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              Docker Compose (bridge: deployment)              │   │
│  │                                                                │   │
│  │  ┌─────────┐   ┌──────────────────────────────────────────┐  │   │
│  │  │  Nginx   │──▶│  vortex-app (Node 18)                    │  │   │
│  │  │  :80     │   │  ├── React Frontend :3000                 │  │   │
│  │  │          │   │  ├── Express Backend :5000                 │  │   │
│  │  │          │   │  └── Docker Socket Mounted                │  │   │
│  │  └─────────┘   └──────────────────────────────────────────┘  │   │
│  │                              │                                  │   │
│  │                    docker run -d                                │   │
│  │                              ▼                                  │   │
│  │                 ┌──────────────────────┐                       │   │
│  │                 │  Build Server         │                       │   │
│  │                 │  (ubuntu:focal)       │                       │   │
│  │                 │  Node 20 + Java 11    │ ──▶ S3 Upload       │   │
│  │                 └──────────┬───────────┘                       │   │
│  │                            │ Kafka Producer                    │   │
│  │                            ▼                                    │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐                         │   │
│  │  │kafka-1  │ │kafka-2  │ │kafka-3  │  KRaft Mode              │   │
│  │  │:19092   │ │:29092   │ │:39092   │  (No ZooKeeper)          │   │
│  │  └─────────┘ └─────────┘ └─────────┘                         │   │
│  │                            │                                    │   │
│  │                            ▼                                    │   │
│  │              ┌──────────────────────┐                          │   │
│  │              │  ClickHouse :8123     │                          │   │
│  │              │  Kafka Engine → MV    │                          │   │
│  │              │  → MergeTree table   │                          │   │
│  │              └──────────────────────┘                          │   │
│  │                                                                │   │
│  │              ┌──────────────────────┐                          │   │
│  │              │  MongoDB :27017      │                          │   │
│  │              │  Users + Deployments │                          │   │
│  │              └──────────────────────┘                          │   │
│  │                                                                │   │
│  │              ┌──────────────────────┐                          │   │
│  │              │  Redpanda Console    │                          │   │
│  │              │  :8080 (Kafka UI)    │                          │   │
│  │              └──────────────────────┘                          │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
          ▲                                              │
          │ SSH (Ansible)                                ▼
    GitHub Actions CD                              AWS S3 Bucket
    (ci.yml + cd.yml)                        (eshwar-vortex-storage)
```

### Key Architecture Decisions
| Decision | Rationale |
|----------|-----------|
| Single EC2 instance | Cost-effective for a portfolio project; all services co-located |
| Docker Compose, not K8s | Simpler orchestration for single-host deployment |
| 3-node Kafka KRaft | Fault-tolerant log streaming without ZooKeeper overhead |
| ClickHouse + Kafka Engine | Zero-code ingestion pipeline; materialized views auto-populate |
| Docker socket mounting | Enables the app container to spawn build containers on the host |
| Nginx reverse proxy | Single entry point, security headers, gzip compression |
| Elastic IP | Stable public IP survives EC2 stop/start cycles |

### Counter Questions
1. "Why all services on one EC2 instance? Isn't that a single point of failure?"
2. "How would you split this into microservices?"
3. "What's the memory requirement? Can a t3.micro handle this?"
4. "Why not use ECS/EKS instead of Docker Compose?"
5. "How does the build container reach Kafka if it's spawned separately?"

### Best Follow-up Answer
"The build container reaches Kafka because it's explicitly joined to the `deployment` Docker bridge network via `--network deployment` in the `docker run` command. This gives it DNS resolution for `kafka-1:19092`. The t3.large is necessary because Kafka (3 brokers × 256MB heap) and ClickHouse together need ~2GB RAM minimum, plus headroom for concurrent builds."

### Confidence Level Required: 🟢 HIGH
### Difficulty: ⭐⭐⭐⭐
### Expected Duration: 5-7 minutes

---

## Q12. "Explain the request lifecycle when a user logs in."

### Ideal Answer
1. User submits email + password on the Auth page
2. Frontend dispatches `loginStart()` Redux action (sets `loading: true`)
3. POST `/api/auth/login` via Axios through Vite's dev proxy or Nginx in production
4. Express receives request → hits `loginUser` controller in [auth.controller.js](file:///Users/eshwarsai/Desktop/Vortex/backend/controllers/auth.controller.js#L47-L75)
5. Looks up user by email in MongoDB
6. Compares password hash using `bcrypt.compare()`
7. On success: generates JWT with `{ id: user._id }`, expiry `7d`, signed with `JWT_SECRET`
8. Returns `{ message, token, user }` (password excluded via destructuring)
9. Frontend dispatches `loginSuccess({ user, token })` — stores in Redux + persisted to localStorage via `redux-persist`
10. `loginTime` is set to `Date.now()` for the `AutoLogout` component's 24-hour expiry check
11. Router redirects to `/home`

### Counter Questions
1. "Why 7-day token expiry? Isn't that too long?"
2. "How do you handle token refresh?"
3. "Where is the JWT verified on subsequent requests?"
4. "What happens if someone steals the JWT from localStorage?"
5. "Why bcryptjs and not argon2?"

---

## Q13. "How does the deployment lifecycle work end-to-end?"

*(Covered comprehensively in Q6 — refer candidates back. In interview, use the same flow.)*

---

## Q14. "How does Kafka fit into the architecture?"

### Ideal Answer
"Kafka serves as a **durable, decoupled log buffer** between the build server (producer) and ClickHouse (consumer). Without Kafka, the build server would need a direct connection to ClickHouse, creating tight coupling. Kafka provides:
1. **Decoupling**: Build servers don't need to know about ClickHouse
2. **Durability**: Logs survive if ClickHouse is temporarily down (replication factor 3, min ISR 2)
3. **Ordering**: Logs within a deployment arrive in order (single partition)
4. **Backpressure**: Kafka absorbs bursts during concurrent builds

The 3-node KRaft cluster eliminates ZooKeeper dependency. Each node runs both broker and controller roles. The `build-logs` topic has 1 partition and replication factor 1 (in production, I'd increase both)."

### Counter Questions
1. "Why 1 partition? Doesn't that limit throughput?"
2. "Your replication factor is 1 but offsets topic has RF=3. Why the inconsistency?"
3. "What happens if all 3 Kafka nodes crash simultaneously?"
4. "Why not Redis Streams instead of Kafka for this use case?"
5. "Why not just write logs directly to ClickHouse?"

---

## Q15. "How does ClickHouse fit into the architecture?"

### Ideal Answer
"ClickHouse serves as the **persistent, queryable log store**. It uses a 3-table pipeline:

1. **`log_queue`** — Kafka Engine table that connects to the `build-logs` topic as consumer group `clickhouse-consumer`, reading `JSONEachRow` format
2. **`kafka_queue`** — Materialized View that transforms data and inserts into the final table, adding `created_at` timestamp and `log_uuid`
3. **`build_logs`** — MergeTree table ordered by `created_at` for efficient time-range queries

This is a zero-code ingestion pipeline. As soon as a message hits Kafka, ClickHouse automatically consumes it, transforms it through the materialized view, and stores it permanently. The backend queries this table when the frontend polls for logs."

### Counter Questions
1. "Why ClickHouse over Elasticsearch for log storage?"
2. "What's the MergeTree engine and why did you choose it?"
3. "How does the Kafka Engine table differ from a regular table?"
4. "What happens if the materialized view has an error?"
5. "How would you handle log retention / TTL?"

---

## Q16-Q25. Architecture Questions Summary

| # | Question | Key Answer Points |
|---|----------|------------------|
| Q16 | "Why MongoDB?" | Document model fits flexible deployment metadata; Mongoose ORM; quick iteration |
| Q17 | "Why Nginx?" | Reverse proxy for API + frontend, gzip compression, security headers, upstream load balancing ready |
| Q18 | "Why Docker?" | Build isolation, reproducible environments, service orchestration |
| Q19 | "Why Terraform?" | Declarative infrastructure, state tracking, reproducible AWS provisioning |
| Q20 | "Why Ansible?" | SSH-based configuration management, Jinja2 templates for env files, idempotent deployments |
| Q21 | "Why not serverless?" | Need long-running build processes (Lambda 15-min limit), Docker-in-Docker, persistent Kafka |
| Q22 | "Why not Kubernetes?" | Over-engineered for single-host; Docker Compose is simpler for this scale |
| Q23 | "Why KRaft not ZooKeeper?" | ZooKeeper adds operational overhead; KRaft is the Kafka 3.x native consensus |
| Q24 | "Why Express not Fastify?" | Mature ecosystem, middleware richness, team familiarity |
| Q25 | "Why Redux not Context?" | Persistence via redux-persist, DevTools, structured state management |

---

# SECTION 3 — FRONTEND

---

## Q26. "Why did you choose React with Vite over Next.js?"

### Ideal Answer
"Vortex's frontend is a pure SPA — it doesn't need server-side rendering or static generation. Vite provides instant HMR, native ESM-based dev server, and faster builds than webpack. Using Next.js would add unnecessary complexity (file-based routing, API routes, server components) for what is essentially a client-rendered dashboard application. Vite's proxy configuration in `vite.config.js` handles API forwarding during development."

### Deep Explanation
From [vite.config.js](file:///Users/eshwarsai/Desktop/Vortex/frontend/vite.config.js):
- Dev server on port 3000 with `host: true` (listens on all interfaces for Docker)
- Proxy `/api` requests to `http://localhost:5000` (Express backend)
- TailwindCSS via `@tailwindcss/vite` plugin

### Counter Questions
1. "But Vite doesn't give you SSR. What if you need SEO?"
2. "Why not use esbuild directly?"
3. "How does HMR work in Vite compared to Webpack?"
4. "What's the production build size?"
5. "How do you handle environment variables in Vite vs CRA?"

---

## Q27. "Explain your routing and protected routes implementation."

### Ideal Answer
"I use React Router v6 with a custom `PrivateRoute` component that checks Redux state for authentication:

```jsx
const PrivateRoute = ({ element }) => {
    const { user } = useSelector((state) => state.user);
    return user ? element : <Navigate to="/" replace />;
};
```

Routes are organized as:
- `/` — WelcomePage (redirects to `/home` if logged in)
- `/auth` — Login/Register page
- `/home` — Repository selection (protected)
- `/dashboard` — Deployment history (protected)
- `/account-settings` — Profile management (protected)
- `/deploy/:username/:repo` — Build & deploy page (protected)
- `*` — Fallback 404 page (protected)

The `AppLayout` component conditionally hides Header/Footer on `/auth` and `/` routes."

### Counter Questions
1. "Why check Redux state instead of JWT token validity?"
2. "What if the JWT expires while the user is on a protected page?"
3. "Why `replace` in the Navigate component?"
4. "How does `PrivateRoute` handle loading states?"
5. "Your PrivateRoute doesn't verify the token with the backend. Is that secure?"

### Best Follow-up Answer
"You're right — the PrivateRoute only checks if `user` exists in Redux state, not if the JWT is actually valid. An expired or tampered token would still show protected pages until the next API call fails. In production, I'd add a token validation middleware that checks expiry on the client side and optionally pings a `/api/auth/verify` endpoint."

---

## Q28. "How does your Redux state management work?"

### Ideal Answer
"I use Redux Toolkit with `redux-persist` for state persistence across page refreshes. The store has a single `user` slice with states: `user`, `token`, `loading`, `error`, `loginTime`.

Key design decisions:
1. **`redux-persist`** with `localStorage` backend, whitelisted to only persist the `user` slice
2. **Serializable check** ignores redux-persist actions (FLUSH, REHYDRATE, etc.) to prevent warnings
3. **`loginTime`** is set on login for the `AutoLogout` component's 24-hour session expiry
4. **Multiple reset actions**: `logout` resets to `initialState`, `logoutSuccess` nullifies individual fields, `resetUser` also resets to initial state

The `AutoLogout` component calculates remaining session time on mount and sets a `setTimeout` to dispatch `logout` when 24 hours have elapsed."

### Counter Questions
1. "Why redux-persist instead of just using localStorage directly?"
2. "What happens on private browsing where localStorage might be unavailable?"
3. "Why store the token in Redux/localStorage instead of httpOnly cookies?"
4. "How do you handle rehydration race conditions?"
5. "What's the difference between `logout` and `logoutSuccess` in your slice?"

---

## Q29-Q35. Frontend Questions Summary

| # | Question | Key Points |
|---|----------|------------|
| Q29 | "How does Axios API service work?" | Centralized `axios.create()` with `baseURL` from `VITE_API_URL` env var |
| Q30 | "How do you handle environment variables in Vite?" | `import.meta.env.VITE_*` prefix; `.env` files for different environments |
| Q31 | "Explain the Deploy page data flow" | Fetch repos via `/api/github`, select repo/branch, generate deploymentId, POST `/api/deploy/start`, poll `/api/logs/:id` |
| Q32 | "How does AutoLogout work?" | Uses `useEffect` with `loginTime` from Redux, sets `setTimeout` for 24h, clears on unmount |
| Q33 | "Why BrowserRouter not HashRouter?" | Clean URLs, proper history API, server-side catch-all needed |
| Q34 | "How do you optimize React performance?" | Vite code splitting, lazy loading potential, minimal re-renders via proper selector usage |
| Q35 | "How does the frontend talk to the backend in production?" | Through Nginx reverse proxy at `/api/` path, no CORS issues since same origin |

---

# SECTION 4 — BACKEND

---

## Q36. "Walk me through your Express server setup."

### Ideal Answer
"The server in [server.js](file:///Users/eshwarsai/Desktop/Vortex/backend/server.js) follows a layered setup:

1. **Security Layer**: Helmet for HTTP security headers, CORS with configurable origin
2. **Rate Limiting**: 5000 requests per 15 minutes per IP on `/api/*` routes — intentionally high to accommodate live log polling
3. **Routing Layer**: 5 route modules — auth, git, deploy, logs, user
4. **Health Check**: `/health` endpoint returns MongoDB connection status and timestamp
5. **Error Handling**: Centralized error handler catches unhandled errors, logs stack trace, returns 500
6. **Graceful Shutdown**: Handles SIGTERM/SIGINT — closes HTTP server first, then MongoDB connection
7. **Trust Proxy**: `app.set('trust proxy', 1)` — required because Express sits behind Nginx, so rate limiter uses `X-Forwarded-For` instead of proxy IP"

### Counter Questions
1. "Why `trust proxy: 1` specifically and not `true`?"
2. "Your rate limit is 5000. How would a real DDoS affect this?"
3. "You don't have request body size limits. What happens with a 100MB POST?"
4. "Where's your authentication middleware for protected routes?"
5. "Why separate route files instead of one `routes/index.js`?"
6. "Your graceful shutdown doesn't wait for in-flight requests. How would you fix that?"

### Best Follow-up Answer
"For `trust proxy: 1`, it means trust exactly one proxy hop (Nginx). Setting it to `true` would trust any proxy in the chain, which is dangerous. For request body limits, Express defaults to 100KB for `express.json()`, but I should add explicit `{ limit: '10mb' }` for deployment payloads. For graceful shutdown, I'd add a `server.close()` callback that uses `setImmediate()` to drain in-flight connections with a configurable timeout."

---

## Q37. "Explain your authentication flow in detail."

### Ideal Answer
"Registration flow ([auth.controller.js](file:///Users/eshwarsai/Desktop/Vortex/backend/controllers/auth.controller.js#L6-L43)):
1. Validate all required fields (username, fullname, password, githubProfile, email)
2. Check for duplicate email AND duplicate username separately
3. Hash password with bcrypt, salt rounds = 10
4. Clean GitHub profile URL — strip `https://github.com/` prefix and trailing `/`
5. Save user to MongoDB

Login flow:
1. Find user by email
2. Compare password with bcrypt
3. Generate JWT with payload `{ id: user._id }`, signed with `JWT_SECRET`, expiry `7d`
4. Return token + user data (password excluded via destructuring: `const { password: _, ...userData } = user._doc`)

Key detail: I use `user._doc` (Mongoose raw document) for destructuring because the Mongoose document object includes getters/setters that would include the password field."

### Counter Questions
1. "Salt rounds of 10 — is that secure enough?"
2. "Why not use `user.toJSON()` with a transform to strip password?"
3. "Where do you verify the JWT on protected API routes?"
4. "Why `_doc` and not `toObject()`?"
5. "What if someone registers with a GitHub profile they don't own?"
6. "You check email and username in separate queries. Why not use `$or`?"

### Best Follow-up Answer
"Salt rounds of 10 is the minimum recommendation. At 10, bcrypt takes ~100ms per hash. I'd increase to 12 for production. For the duplicate check, using `$or` in a single query would be more efficient but gives less specific error messages. The tradeoff is 2 DB queries vs. a better user experience. In high-traffic production, I'd use a unique index error handler instead."

---

## Q38. "Explain your deploy controller — this is the most critical code."

### Ideal Answer
"The [deploy.controller.js](file:///Users/eshwarsai/Desktop/Vortex/backend/controllers/deploy.controller.js) has 4 endpoints:

1. **`deployProject` (POST /api/deploy/start)** — The core deployment trigger:
   - Uses `execSync` (synchronous!) to check/build Docker images
   - Uses `exec` (async) for `docker wait` to monitor container
   - Passes AWS credentials, Kafka broker address, S3 bucket, and user env vars to the container
   - Returns 200 immediately — deployment is async

2. **`createDeployment` (POST /api/deploy/create)** — Saves deployment metadata:
   - Upserts (findOne then save) instead of `findOneAndUpdate`
   - Includes a URL replacement regex for S3 bucket migration

3. **`getDeploymentByRepoAndUser` (GET /api/deploy/get)** — Checks if a repo has been deployed
4. **`getDeploymentsByUser` (GET /api/deploy/getdeploy)** — Lists all deployments for a user's dashboard"

### Counter Questions
1. "Using `execSync` blocks the event loop. How does that affect concurrent requests?"
2. "You pass AWS credentials as Docker environment variables. They're visible in `docker inspect`. How do you mitigate this?"
3. "What if `docker run` succeeds but the container immediately crashes?"
4. "You don't have any deployment status tracking. How does the frontend know if deployment succeeded?"
5. "The URL replacement regex is hardcoded. Why?"

### Best Follow-up Answer
"Using `execSync` for `docker image inspect` is a conscious tradeoff — it's fast (<100ms) and only happens once per deployment. But `execSync` for `docker build` (the fallback) could block for minutes. In production, I'd use the Docker Engine API via `dockerode` npm package, which provides async, non-blocking operations. For credentials, I'd use Docker secrets or AWS IAM instance profiles with STS temporary credentials instead of passing them as env vars."

---

## Q39-Q47. Backend Questions Summary

| # | Question | Key Points |
|---|----------|------------|
| Q39 | "Explain your middleware architecture" | Helmet, CORS, rate limiter, express.json — but notably missing JWT verification middleware |
| Q40 | "How does the GitHub integration work?" | Octokit SDK, optional auth token, `listForUser`, `listBranches`, URL cleaning |
| Q41 | "How do log queries work?" | Direct ClickHouse HTTP client, SQL query with string interpolation (⚠️ injection risk!) |
| Q42 | "Why Express 5?" | `package.json` shows `express@^5.1.0` — async error handling built-in, path route improvements |
| Q43 | "Explain graceful shutdown" | SIGTERM/SIGINT handlers, close HTTP server first, then MongoDB connection, `process.exit(0)` |
| Q44 | "Where's your input validation?" | Manual validation in controllers; should use Joi or Zod |
| Q45 | "How do you handle async errors?" | try-catch blocks in async controllers; centralized error handler as fallback |
| Q46 | "Explain your folder structure" | controllers/, models/, routes/, middlewares/ — standard MVC-ish pattern |
| Q47 | "Why ES modules (`type: module`)?" | Modern syntax, top-level await support, tree-shaking, native Node.js support |

---

# SECTION 5 — DATABASE (MongoDB)

---

## Q48. "Why did you choose MongoDB over PostgreSQL?"

### Ideal Answer
"For Vortex, MongoDB was the right choice because:
1. **Schema flexibility**: Deployment metadata varies — some deployments have env vars, some have logs, some don't. MongoDB's document model handles this naturally.
2. **Rapid prototyping**: Mongoose provides quick schema definition without migrations.
3. **Embedded logs**: The deployment schema has an embedded `logs` array, which is a natural document pattern.
4. **Read-heavy workload**: Dashboard queries are simple `find()` operations — no complex JOINs needed.

However, PostgreSQL would be better if I needed:
- ACID transactions across multiple tables
- Complex relational queries (e.g., user → teams → projects → deployments)
- Better consistency guarantees"

### Counter Questions
1. "Your deployment model has an embedded `logs` array. What happens when it grows to 10,000 entries?"
2. "Why Mongoose and not the native MongoDB driver?"
3. "What indexes exist on your collections? Did you create any?"
4. "How would you handle a scenario where you need to join users and deployments?"
5. "MongoDB has a 16MB document limit. How does that affect your logs array?"

### Best Follow-up Answer
"Great point about the 16MB limit. The deployment model's `logs` array is actually a legacy field — the real logs are stored in ClickHouse. The embedded array was from an earlier design iteration and could be removed. For indexes, Mongoose auto-creates indexes for `unique: true` fields (`username`, `email`, `deploymentId`). For production, I'd add compound indexes like `{ username: 1, repoName: 1 }` for the deployment lookup query."

---

## Q49-Q57. MongoDB Questions

| # | Question | Key Points |
|---|----------|------------|
| Q49 | "Explain your User schema" | username (unique), fullname, email (unique), githubProfile, password; timestamps auto-generated |
| Q50 | "Explain your Deployment schema" | deploymentId (unique), repoName, branch, username, logs[] (embedded), url; timestamps |
| Q51 | "What indexes do you have?" | Auto-indexes on unique fields; should add compound index on `{ username: 1 }` for getDeploymentsByUser |
| Q52 | "How would you handle aggregation?" | Pipeline: `$match` by user → `$group` by repoName → `$sort` by date for analytics |
| Q53 | "How do you handle transactions?" | Not currently needed; single-document operations are atomic in MongoDB |
| Q54 | "How would you scale MongoDB?" | Replica set for HA, sharding by username for horizontal scaling |
| Q55 | "How do you back up MongoDB?" | `backup.sh` uses `docker exec mongodb mongodump --archive --gzip` |
| Q56 | "Why not use Prisma?" | Prisma is SQL-focused; Mongoose is the standard for MongoDB in Node.js |
| Q57 | "What's your connection pooling strategy?" | Mongoose defaults (5 connections); would increase for production |

---

# SECTION 6 — APACHE KAFKA

---

## Q58. "Explain your Kafka cluster setup in detail."

### Ideal Answer
"I run a 3-node Kafka cluster in KRaft mode (no ZooKeeper). From [docker-compose.yml](file:///Users/eshwarsai/Desktop/Vortex/services/docker-compose.yml):

**Node Configuration:**
- Each node has dual roles: `broker,controller` (KAFKA_PROCESS_ROLES)
- Controller quorum: `1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093`
- Each node has 256MB max heap, 128MB initial heap (`KAFKA_HEAP_OPTS`)
- Ports: kafka-1 (19092), kafka-2 (29092), kafka-3 (39092)

**Cluster Settings:**
- `offsets.topic.replication.factor: 3` — Internal offsets topic is fully replicated
- `transaction.state.log.replication.factor: 3` — Transaction metadata fully replicated
- `transaction.state.log.min.isr: 2` — At least 2 in-sync replicas for transactions
- `auto.create.topics.enable: false` — Topics must be explicitly created
- `group.initial.rebalance.delay.ms: 0` — Immediate rebalancing (for dev speed)

**Topic:** `build-logs` with 1 partition, replication factor 1 (created by topic-init container)

**Monitoring:** Redpanda Console (v2.5.2) on port 8080 provides a web UI for topic inspection."

### Counter Questions
1. "Your topic has replication factor 1 but your offsets topic has RF 3. If the single replica broker dies, what happens?"
2. "Why KRaft over ZooKeeper? What's the actual operational difference?"
3. "You have 3 brokers but 1 partition. What's the point of 3 brokers then?"
4. "How does the Kafka producer in the build server handle broker failures?"
5. "What's the message format? How is each log structured?"
6. "You're using `kafkajs` — what are its limitations vs. `confluent-kafka`?"
7. "Why `auto.create.topics.enable: false`?"
8. "What happens to unacknowledged messages if the producer crashes mid-build?"

### Best Follow-up Answer
"The 3 brokers with 1 partition means only one broker handles the `build-logs` topic while the other two handle internal topics (offsets, transactions). This is a tradeoff for simplicity. In production, I'd increase the topic to 3 partitions with RF 3, keyed by `deployment_id` for ordering guarantees per deployment. For producer failures, `kafkajs` defaults to `acks: -1` (all replicas) but with RF 1 that means only one acknowledgment. If the producer crashes, any messages in the local buffer are lost — I'd add `enableIdempotence: true` for exactly-once semantics."

---

## Q59. "Explain the Kafka producer in your build server."

### Ideal Answer
From [script.js](file:///Users/eshwarsai/Desktop/Vortex/build-server/script.js#L27-L53):

```javascript
const kafka = new Kafka({
    clientId: 'deployment',
    brokers: [KAFKA_BROKER]  // Single broker connection string
});

const producer = kafka.producer();

async function publishLog(logMessage, logLevel = 'info') {
    await producer.send({
        topic: KAFKA_TOPIC,
        messages: [{
            key: 'log',
            value: JSON.stringify({
                deployment_id: DEPLOYMENT_ID,
                log_message: logMessage,
                log_level: logLevel
            })
        }]
    });
}
```

Key observations:
1. **Single broker connection**: Only connects to `kafka-1:19092` — if kafka-1 is down, build logs are lost
2. **Static key `'log'`**: All messages go to the same partition (since there's only 1 partition anyway)
3. **Await on every send**: Each log line blocks until Kafka acknowledges — could batch for performance
4. **No error handling on `producer.send()`**: If Kafka is unreachable, the build fails silently
5. **No `producer.disconnect()`**: Producer connection may leak when the container exits

### Counter Questions
1. "Why `key: 'log'` instead of `key: DEPLOYMENT_ID`?"
2. "What's the impact of awaiting every send call?"
3. "How would you batch logs for better throughput?"
4. "What if the producer can't connect to Kafka at startup?"
5. "You only connect to one broker. What about metadata discovery?"

---

## Q60-Q69. Kafka Deep Dive

| # | Question | Key Points |
|---|----------|------------|
| Q60 | "What are consumer groups?" | ClickHouse uses `clickhouse-consumer` group; enables parallel consumption |
| Q61 | "Explain partitions vs topics" | Topic is a category (build-logs), partitions enable parallelism within a topic |
| Q62 | "What are offsets?" | Position marker per partition per consumer group; ClickHouse auto-commits |
| Q63 | "How does leader election work in KRaft?" | Raft consensus among controller voters; quorum of 2/3 needed |
| Q64 | "What's min.insync.replicas?" | Set to 2 for transactions — 2 of 3 replicas must acknowledge before commit |
| Q65 | "Explain exactly-once semantics" | Requires idempotent producer + transactional API; not currently implemented |
| Q66 | "What's at-least-once?" | Producer retries on failure; consumer may re-read after crash; duplicates possible |
| Q67 | "How do you handle ordering?" | Single partition guarantees FIFO; with multiple partitions, order only within partition |
| Q68 | "What's the benchmark setup?" | `benchmark-kafka.sh` — 1M records × 100 bytes, 6 partitions, RF 3, acks=all |
| Q69 | "What happens if a consumer lags?" | ClickHouse Kafka Engine auto-catches up; `kafka_skip_broken_messages = 1` ignores malformed messages |

---

# SECTION 7 — CLICKHOUSE

---

## Q70. "Why ClickHouse for log storage?"

### Ideal Answer
"ClickHouse is a columnar OLAP database optimized for analytical queries over large datasets. For build logs, the query pattern is:
- Write-heavy: Thousands of log lines per build
- Read pattern: `SELECT * WHERE deployment_id = X ORDER BY created_at`
- Append-only: Logs are never updated

ClickHouse's MergeTree engine is perfect for this — it's optimized for sequential writes and range scans on the sort key (`created_at`). Compared to MongoDB for logs:
- 10-100x better compression (columnar storage)
- 10-100x faster for analytical queries
- Native Kafka integration via the Kafka Engine

MongoDB is still used for user profiles and deployment metadata where random read/write access patterns dominate."

### Counter Questions
1. "Why not Elasticsearch for logs?"
2. "ClickHouse isn't great for point lookups. How does `WHERE deployment_id = X` perform?"
3. "You're using `created_at` as the sort key. Why not `deployment_id, created_at`?"
4. "What's the Kafka Engine vs. a regular consumer application?"
5. "How do materialized views work in ClickHouse?"

---

## Q71. "Explain your ClickHouse schema."

### Ideal Answer
From [init-table.sql](file:///Users/eshwarsai/Desktop/Vortex/services/init-table.sql):

```sql
-- Table 1: Kafka Engine (virtual consumer)
CREATE TABLE logs.log_queue (
    deployment_id String,
    log_message String,
    log_level String
) ENGINE = Kafka('kafka-1:19092,kafka-2:19092,kafka-3:19092',
                 'build-logs', 'clickhouse-consumer', 'JSONEachRow')
SETTINGS kafka_skip_broken_messages = 1;

-- Table 2: Persistent storage
CREATE TABLE logs.build_logs (
    created_at DateTime64(3, 'Asia/Kolkata'),
    log_uuid UUID DEFAULT generateUUIDv4(),
    deployment_id String,
    log_message String,
    log_level String
) ENGINE = MergeTree ORDER BY created_at;

-- Table 3: Auto-ingestion pipeline
CREATE MATERIALIZED VIEW logs.kafka_queue TO logs.build_logs AS
SELECT
    toTimezone(now(), 'Asia/Kolkata') AS created_at,
    generateUUIDv4() AS log_uuid,
    deployment_id, log_message, log_level
FROM logs.log_queue;
```

**Key Design Decisions:**
1. **3-table pattern**: Kafka Engine → Materialized View → MergeTree (standard ClickHouse Kafka ingestion pattern)
2. **`DateTime64(3, 'Asia/Kolkata')`**: Millisecond precision with IST timezone
3. **`generateUUIDv4()`**: Unique ID per log entry for deduplication
4. **`ORDER BY created_at`**: Optimizes time-range queries
5. **`kafka_skip_broken_messages = 1`**: Skip malformed messages instead of crashing"

### Counter Questions
1. "Why not `ORDER BY (deployment_id, created_at)` for better query performance?"
2. "What happens to the Kafka Engine table if Kafka is down?"
3. "How does ClickHouse handle backfilling if it was down during a build?"
4. "What's the compression ratio you're seeing?"
5. "How would you add TTL for log retention?"

---

# SECTION 8 — DOCKER

---

## Q72. "Explain your Docker setup. You have multiple Dockerfiles."

### Ideal Answer
"Vortex has 3 Dockerfiles serving different purposes:

1. **Root Dockerfile** ([Dockerfile](file:///Users/eshwarsai/Desktop/Vortex/Dockerfile)) — Main application container:
   - Base: `node:18`
   - Installs `docker.io` inside the container (for Docker-in-Docker)
   - Installs `npm-run-all` globally
   - Copies frontend + backend + root package.json
   - Runs `npm install` for root, frontend, and backend
   - CMD: `npm-run-all --parallel start-frontend start-backend`
   - Exposes ports 3000 (Vite) and 5000 (Express)

2. **Build Server Dockerfile** ([build-server/Dockerfile](file:///Users/eshwarsai/Desktop/Vortex/build-server/Dockerfile)) — Ephemeral build container:
   - Base: `ubuntu:focal`
   - Installs: curl, git, openjdk-11-jdk, maven, jq, Node.js 20
   - Copies: `main.sh`, `script.js`, `package*.json`
   - ENTRYPOINT: `/home/app/main.sh`
   - Purpose: Clone repo → build → upload to S3

3. **Nginx Dockerfile** ([nginx/Dockerfile](file:///Users/eshwarsai/Desktop/Vortex/nginx/Dockerfile)) — Reverse proxy:
   - Base: `nginx:1.25-alpine`
   - Copies custom `nginx.conf`
   - Exposes port 80"

### Counter Questions
1. "Your main Dockerfile isn't multi-stage. Why?"
2. "You install `docker.io` inside a container. That's Docker-in-Docker. What are the risks?"
3. "Why `ubuntu:focal` for the build server instead of a lighter image?"
4. "Your Dockerfile runs `npm install` 3 times. How would you optimize this?"
5. "What's the image size for each container?"
6. "Why not use Alpine for the main app container?"

### Best Follow-up Answer
"Multi-stage builds would help — I'd use a builder stage for `npm install` and `npm run build` (frontend), then copy only the `dist/` folder and backend source to a slim runtime image. For the build server, `ubuntu:focal` is necessary because it needs a full userland (git, java, node, maven) to handle diverse project types. Alpine would break some npm packages that depend on glibc."

---

## Q73. "Explain your Docker Compose setup."

### Ideal Answer
"[docker-compose.yml](file:///Users/eshwarsai/Desktop/Vortex/services/docker-compose.yml) orchestrates 8 services on a single bridge network called `deployment`:

| Service | Image | Ports | Health Check | Restart |
|---------|-------|-------|-------------|---------|
| kafka-1 | apache/kafka:3.8.1 | 19092 | — | unless-stopped |
| kafka-2 | apache/kafka:3.8.1 | 29092 | — | unless-stopped |
| kafka-3 | apache/kafka:3.8.1 | 39092 | — | unless-stopped |
| topic-init | apache/kafka:3.8.1 | — | — | no |
| console | redpanda console v2.5.2 | 8080 | — | always |
| clickhouse | clickhouse-server:23.4 | 8123, 9000 | wget ping | unless-stopped |
| mongodb | mongo:6.0 | 27017 | mongosh ping | unless-stopped |
| application | Custom (Dockerfile) | 3005→3000, 5005→5000 | curl /health | unless-stopped |
| nginx | Custom (nginx/) | 80 | — | unless-stopped |

**Key Design Decisions:**
1. **Dependency ordering**: Application depends on kafka-1 (started), clickhouse (healthy), mongodb (healthy)
2. **Docker socket mount**: `/var/run/docker.sock` mapped into application container
3. **Named volumes**: `kafka1_data`, `kafka2_data`, `kafka3_data`, `mongo_data` for persistence
4. **topic-init container**: One-shot container that creates the `build-logs` topic, then exits (`restart: no`)
5. **ClickHouse ulimits**: `nofile: 262144` for handling many open files"

### Counter Questions
1. "Why named volumes instead of bind mounts?"
2. "Kafka nodes don't have health checks. How do you know they're ready?"
3. "The topic-init container depends on all 3 Kafka nodes being 'started', but not 'healthy'. What if Kafka isn't ready yet?"
4. "What happens if you `docker compose down` — is data preserved?"
5. "Why separate port mappings (3005→3000)?"
6. "You mount the Docker socket. Can the application container compromise the host?"

---

## Q74-Q83. Docker Deep Dive

| # | Question | Key Points |
|---|----------|------------|
| Q74 | "Explain Docker layers and caching" | Each Dockerfile instruction = layer; COPY before RUN npm install breaks cache |
| Q75 | "How do Docker networks work in your setup?" | Bridge network `deployment`; DNS resolution by container name |
| Q76 | "What happens when the Docker daemon crashes?" | All containers stop; `unless-stopped` restarts them on daemon restart |
| Q77 | "Explain the build server container lifecycle" | Created → Running (clone+build+upload) → Exited; not auto-removed |
| Q78 | "How do you handle Docker logs?" | `docker logs <container>`; in CD pipeline, `--tail=100` for diagnostics |
| Q79 | "Why not use Docker Swarm?" | Single-host deployment; Swarm adds complexity without benefit |
| Q80 | "Explain volume persistence" | Named volumes stored in `/var/lib/docker/volumes/`; survive `down` but not `down -v` |
| Q81 | "How would you optimize image sizes?" | Multi-stage builds, Alpine base, `.dockerignore`, minimize layers |
| Q82 | "What's the difference between CMD and ENTRYPOINT?" | ENTRYPOINT (build-server) = always runs; CMD (nginx) = default, overridable |
| Q83 | "How do health checks work in Docker Compose?" | `healthcheck` block with test command, interval, timeout, retries |

---

# SECTION 9 — TERRAFORM

---

## Q84. "Walk me through your Terraform infrastructure."

### Ideal Answer
"My Terraform configuration in [infra/terraform/](file:///Users/eshwarsai/Desktop/Vortex/infra/terraform) provisions the complete AWS infrastructure:

**Files:**
- `provider.tf` — AWS provider, region from variable
- `versions.tf` — Terraform ≥ 1.6.0, AWS provider ~> 5.0
- `variables.tf` — 11 configurable variables with sane defaults
- `networking.tf` — VPC, subnet, internet gateway, route table
- `security.tf` — Security group with 21 ingress/egress rules
- `ec2.tf` — EC2 instance, Elastic IP, EIP association
- `iam.tf` — IAM role, SSM policy, instance profile
- `outputs.tf` — Instance ID, public IP, elastic IP, DNS, SSH command
- `user_data.sh` — Bootstrap script (Docker, Git installation)

**Network Architecture:**
```
VPC (10.0.0.0/16)
  └── Public Subnet (10.0.1.0/24) in ap-south-1a
       └── Internet Gateway → Route Table (0.0.0.0/0 → IGW)
            └── EC2 t3.large + Elastic IP
                 └── Security Group (22 rules)
```

**Security Group Strategy:**
- SSH: Restricted to `allowed_ssh_cidr` (defaults to non-routable `192.0.2.0/24`)
- HTTP/HTTPS: Open to `0.0.0.0/0`
- App ports (Kafka, ClickHouse, MongoDB, Console): Restricted to `allowed_app_ports_cidr`
- VPC internal: All app ports also allow traffic from within the VPC CIDR
- Egress: All outbound traffic allowed"

### Counter Questions
1. "Your Terraform state file is in the repo. Why is that dangerous?"
2. "How do you handle state locking?"
3. "What happens if `terraform apply` fails midway?"
4. "Why `lifecycle { ignore_changes = [ami] }`?"
5. "Why `http_tokens = required` in metadata options?"
6. "How would you make this multi-environment (dev/staging/prod)?"

### Best Follow-up Answer
"The state file should NEVER be committed. I'd use an S3 backend with DynamoDB locking:
```hcl
terraform {
  backend \"s3\" {
    bucket         = \"vortex-terraform-state\"
    key            = \"production/terraform.tfstate\"
    region         = \"ap-south-1\"
    dynamodb_table = \"terraform-locks\"
    encrypt        = true
  }
}
```
`http_tokens = required` enforces IMDSv2 (Instance Metadata Service v2), which mitigates SSRF attacks that could steal IAM credentials from the metadata endpoint."

---

## Q85. "Explain your EC2 user_data bootstrap script."

### Ideal Answer
"[user_data.sh](file:///Users/eshwarsai/Desktop/Vortex/infra/terraform/user_data.sh) runs on first EC2 launch:

1. **Swap allocation**: Creates 4GB swap file to prevent OOM during Docker builds
2. **Retry helper**: `retry_cmd()` function with 5 attempts, 5s delay — handles transient apt failures
3. **Prerequisites**: Installs ca-certificates, curl, gnupg, git, software-properties-common
4. **Docker installation**: Official Docker repository, installs docker-ce, docker-compose-plugin, buildx
5. **User permissions**: Adds `ubuntu` user to `docker` group
6. **Validation**: Verifies `docker`, `docker compose`, and `git` are available; exits 1 on failure

Key design decision: The `retry_cmd` function handles the common issue of apt repository timeouts on fresh EC2 instances where the network isn't fully configured yet."

### Counter Questions
1. "How do you debug user_data failures?"
2. "Why 4GB swap and not 2GB?"
3. "This script doesn't install the application. How does the code get there?"
4. "What if the Docker GPG key URL changes?"
5. "Why not use a pre-baked AMI instead of user_data?"

---

## Q86-Q95. Terraform Deep Dive

| # | Question | Key Points |
|---|----------|------------|
| Q86 | "Explain Terraform state" | JSON file tracking real infrastructure; must be stored remotely for teams |
| Q87 | "What are Terraform modules?" | Reusable infrastructure packages; current setup is flat but could be modularized |
| Q88 | "Explain `terraform plan` vs `apply`" | Plan = dry run showing changes; apply = executes changes; always plan before apply |
| Q89 | "What is state drift?" | When real infrastructure differs from state; detect with `terraform plan`, fix with `apply` |
| Q90 | "Why `create_before_destroy` on the security group?" | Prevents downtime — new SG created before old one destroyed |
| Q91 | "Explain the AMI data source" | Dynamic lookup of latest Ubuntu 24.04 AMI from Canonical; overridable via `ami_id` variable |
| Q92 | "Why gp3 volume type?" | Better price/performance than gp2; 3000 IOPS baseline without provisioning |
| Q93 | "Why `encrypted = true` on root volume?" | Compliance requirement; encrypts data at rest using AWS-managed KMS key |
| Q94 | "Explain the IAM role chain" | Role (sts:AssumeRole) → SSM policy attachment → Instance Profile → EC2 |
| Q95 | "How would you add a staging environment?" | Terraform workspaces or separate variable files (`terraform.tfvars.staging`) |

---

# SECTION 10 — ANSIBLE

---

## Q96. "Explain your Ansible deployment pipeline."

### Ideal Answer
"Ansible handles **configuration management and application deployment** after Terraform provisions the infrastructure.

**Structure:**
```
infra/ansible/
├── ansible.cfg       # Config: YAML output, no retry files, host_key_checking off
├── inventory.ini     # Single host: 13.235.26.67 (EC2 Elastic IP)
├── group_vars/all.yml  # Shared variables (repo URL, ports, health endpoint)
├── playbook.yml      # Entry point: runs 'vortex' role on vortex_hosts
└── roles/vortex/
    ├── defaults/main.yml   # Default vars (jwt_secret, aws credentials from env)
    ├── vars/main.yml       # Empty (higher precedence override point)
    ├── tasks/
    │   ├── main.yml        # Orchestrator: repo → env → deploy → flush → health
    │   ├── repository.yml  # Git clone/pull with force
    │   ├── env.yml         # Generate .env from Jinja2 template
    │   ├── deploy.yml      # Docker compose pull → up → build build-server image
    │   └── health.yml      # Container health checks + HTTP endpoint verification
    ├── handlers/main.yml   # Restart Docker services (triggered by env changes)
    └── templates/.env.j2   # Jinja2 template for .env file
```

**Execution Flow:**
1. Clone/update repo from GitHub
2. Generate `.env` from Jinja2 template (triggers handler if changed)
3. `docker compose pull` → `docker compose up -d --build`
4. Build `vortex-build-server:latest` image
5. Flush handlers (restart services if env changed)
6. Wait for vortex-app container to be healthy (12 retries × 10s)
7. Verify `docker compose ps` has no Exit/unhealthy states
8. HTTP health check on `http://localhost:80/health`
9. Verify build-server image exists"

### Counter Questions
1. "Why not use Ansible Vault for secrets?"
2. "You hardcode the IP in inventory. How do you make this dynamic?"
3. "Why `host_key_checking = False`?"
4. "What's the difference between `defaults` and `vars` in a role?"
5. "How does the handler know when to restart?"
6. "Why `force: yes` on git clone?"
7. "Your env template uses `lookup('env', 'AWS_ACCESS_KEY_ID')`. Where do those env vars come from?"

### Best Follow-up Answer
"The env vars come from the GitHub Actions CD workflow, which sets `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` from repository secrets before running `ansible-playbook`. The `lookup('env', ...)` function reads from the runner's environment. For a more secure setup, I'd use Ansible Vault to encrypt secrets at rest and AWS Secrets Manager for runtime secrets."

---

## Q97-Q105. Ansible Questions Summary

| # | Question | Key Points |
|---|----------|------------|
| Q97 | "What is idempotency?" | Running playbook multiple times produces same result; `git clone` with `force: yes` handles this |
| Q98 | "Explain the Jinja2 template" | `.env.j2` generates production `.env` with Ansible variables injected |
| Q99 | "How do handlers work?" | Only fire when `notify` is triggered by a changed task; `meta: flush_handlers` forces execution |
| Q100 | "What's `become: true`?" | Runs with sudo; needed for Docker commands and file permissions |
| Q101 | "Why YAML stdout callback?" | Human-readable output instead of default JSON; paired with timer and profile_tasks |
| Q102 | "How would you test Ansible playbooks?" | Molecule framework, Vagrant, or Docker containers for local testing |
| Q103 | "What's the `changed_when` directive?" | Tells Ansible when a command task actually made changes vs. was idempotent |
| Q104 | "Explain the health check strategy" | 3-layer: Docker health status → compose ps assertions → HTTP health endpoint |
| Q105 | "Why not use Ansible Tower/AWX?" | Overkill for single-host; GitHub Actions provides the orchestration layer |

---

# SECTION 11 — GITHUB ACTIONS CI/CD

---

## Q106. "Explain your CI pipeline."

### Ideal Answer
"[ci.yml](file:///Users/eshwarsai/Desktop/Vortex/.github/workflows/ci.yml) runs on PRs to main and manual dispatch. 6 stages:

1. **Backend checks**: Setup Node 20, `npm ci`, `npm test` (non-blocking)
2. **Frontend checks**: `npm install`, `npm run build` (validates compilation)
3. **Docker checks**: `docker compose config` (syntax) + `docker compose build` (image compilation)
4. **Terraform checks**: `terraform fmt -check` + `terraform init -backend=false` + `terraform validate`
5. **Security - Trivy**: Filesystem scan for vulnerabilities (non-blocking, `exit-code: 0`)
6. **Security - Gitleaks**: Scan for exposed secrets in git history
7. **Security - Snyk**: Dependency vulnerability scan across all Node.js projects"

### Counter Questions
1. "Why is `npm test` non-blocking (`|| echo ...`)?"
2. "Why `npm ci` for backend but `npm install` for frontend?"
3. "Terraform init with `-backend=false` — why?"
4. "Trivy `exit-code: 0` means vulnerabilities don't fail the build. Why?"
5. "How long does this pipeline take?"
6. "Why not run stages in parallel using a matrix?"

---

## Q107. "Explain your CD pipeline."

### Ideal Answer
"[cd.yml](file:///Users/eshwarsai/Desktop/Vortex/.github/workflows/cd.yml) runs on pushes to main and manual dispatch. 5 stages:

1. **SSH Setup**: Write private key from secrets, set permissions (600), ssh-keyscan for known_hosts
2. **SSH Verify**: Echo test on remote host to validate connectivity
3. **Ansible Deploy**: Run playbook with dynamic host/user from secrets + AWS credentials
4. **Health Check**: Retry loop — 12 attempts × 15s delay (3 minutes total) polling `http://{host}/health`
5. **Diagnostics (on failure)**: SSH to host, collect `docker ps -a`, `docker compose ps`, container logs

Key security note: The SSH private key is stored as a GitHub secret (`EC2_PRIVATE_KEY`), written to a file with `chmod 600`, and the host's key is pre-scanned to prevent MITM prompts."

### Counter Questions
1. "Why not use a deployment key instead of a full SSH private key?"
2. "What if the health check passes but the app is partially broken?"
3. "How do you handle rollbacks if the deploy fails?"
4. "Why not use AWS CodeDeploy instead of Ansible over SSH?"
5. "Your diagnostics step runs on `if: failure()`. What if SSH itself is the failure?"

---

## Q108-Q114. CI/CD Questions Summary

| # | Question | Key Points |
|---|----------|------------|
| Q108 | "How do GitHub secrets work?" | Encrypted at rest, masked in logs, scoped to repo/environment |
| Q109 | "What are workflow triggers?" | `push`, `pull_request`, `workflow_dispatch`; can filter by branch |
| Q110 | "How does caching work?" | `actions/setup-node` caches npm via `cache-dependency-path` |
| Q111 | "What are artifacts in GitHub Actions?" | Build outputs preserved between jobs; not used in current pipeline |
| Q112 | "How would you add matrix builds?" | Test across Node 18/20/22, multiple OS versions |
| Q113 | "Why separate CI and CD workflows?" | CI validates PRs (gate), CD deploys merged code (ship) |
| Q114 | "How do you handle concurrent deployments?" | GitHub Actions concurrency groups can prevent simultaneous deploys |

---

# SECTION 12 — AWS INFRASTRUCTURE

---

## Q115-Q130. AWS Questions

| # | Question | Key Points |
|---|----------|------------|
| Q115 | "Why EC2 over Lambda?" | Long-running builds, Docker dependency, persistent services (Kafka, ClickHouse) |
| Q116 | "Why t3.large?" | 8GB RAM needed for Kafka (3×256MB) + ClickHouse + concurrent builds |
| Q117 | "Explain Elastic IP" | Survives EC2 stop/start; prevents DNS/inventory changes |
| Q118 | "Explain your VPC design" | Custom VPC (10.0.0.0/16), single public subnet, IGW, public route table |
| Q119 | "Why not use a NAT gateway?" | All services need public access; no private subnets; NAT adds cost |
| Q120 | "Explain your security group strategy" | Layered: SSH restricted, HTTP open, app ports restricted, VPC internal allowed |
| Q121 | "Why `192.0.2.0/24` as default CIDR?" | RFC 5737 documentation range — non-routable, forces override |
| Q122 | "Explain IAM role for EC2" | Role → SSM policy (for remote management) → instance profile → attached to EC2 |
| Q123 | "How does S3 fit?" | Build artifacts stored under `__outputs/{deployment_id}/`; accessed via S3 URL |
| Q124 | "Why S3 and not EFS/EBS?" | S3 is object storage ideal for static files; globally accessible, cheap |
| Q125 | "What's IMDSv2 and why enforce it?" | Instance Metadata Service v2 requires session tokens; mitigates SSRF attacks |
| Q126 | "How would you add CloudWatch monitoring?" | CloudWatch agent for metrics, alarms for CPU/memory, log group for container logs |
| Q127 | "Why `gp3` volume?" | Baseline 3000 IOPS, 125MB/s throughput; better than gp2 for Docker workloads |
| Q128 | "Why 30GB root volume?" | Docker images, Kafka data, ClickHouse data, build artifacts need space |
| Q129 | "How would you add auto-scaling?" | ASG with launch template; but requires shared state (EFS/RDS instead of local volumes) |
| Q130 | "What's the monthly cost estimate?" | t3.large (~$60/mo) + EIP (free when attached) + S3 (~$1/mo) ≈ ~$65/mo |

---

# SECTION 13 — NGINX

---

## Q131. "Explain your Nginx configuration."

### Ideal Answer
"[nginx.conf](file:///Users/eshwarsai/Desktop/Vortex/nginx/nginx.conf) acts as the edge gateway:

**Performance:**
- `worker_processes auto` — scales to CPU count
- `worker_connections 1024` — per-worker concurrent connections
- `sendfile on`, `tcp_nopush on`, `tcp_nodelay on` — kernel-level optimizations
- `keepalive_timeout 65` — persistent connections

**Compression:**
- gzip level 6 on text, CSS, JSON, JS, XML

**Security Headers:**
- `X-Frame-Options: SAMEORIGIN` — prevents clickjacking
- `X-XSS-Protection: 1; mode=block` — browser XSS filter
- `X-Content-Type-Options: nosniff` — prevents MIME sniffing
- `Referrer-Policy: no-referrer-when-downgrade` — controls referrer header

**Upstreams:**
- `backend_nodes → application:5000` (Express)
- `console_nodes → console:8080` (Redpanda)

**Locations:**
- `/api/` → proxy to backend with WebSocket upgrade headers
- `/health` → proxy to backend health endpoint
- `/console/` → proxy to Redpanda Console
- `/` → static JSON response ('Vortex Production Edge Gateway Online')"

### Counter Questions
1. "Why is there no SSL/TLS configuration?"
2. "Why `server_name _` (catch-all)?"
3. "The root location returns static JSON. How does the frontend get served?"
4. "Why proxy WebSocket headers (`Upgrade`, `Connection`) on API routes?"
5. "How would you add rate limiting at the Nginx level?"
6. "What happens if the backend upstream is down?"

### Best Follow-up Answer
"The frontend is served by Vite's dev server inside the application container (port 3000), but it's not proxied through Nginx in the current setup. In production, I'd build the frontend, serve the static files from Nginx directly, and only proxy API calls to the backend. For SSL, I'd use Let's Encrypt with certbot or AWS Certificate Manager with a load balancer."

---

## Q132-Q136. Nginx Questions

| # | Question | Key Points |
|---|----------|------------|
| Q132 | "How does reverse proxy work?" | Client → Nginx → Backend; Nginx forwards request, sets proper headers |
| Q133 | "What are `X-Real-IP` and `X-Forwarded-For`?" | Pass client's real IP to backend behind proxy |
| Q134 | "How does gzip compression work?" | Compresses response body; `gzip_vary` tells caches to vary by Accept-Encoding |
| Q135 | "Why Alpine for the Nginx image?" | ~5MB image size; Nginx doesn't need full OS |
| Q136 | "How would you add load balancing?" | Add more backend instances to `upstream` block; use `least_conn` or `ip_hash` |

---

# SECTION 14 — SECURITY

---

## Q137. "What security measures does Vortex implement?"

### Ideal Answer

| Layer | Measure | Implementation |
|-------|---------|---------------|
| Transport | Helmet HTTP headers | X-Frame-Options, X-XSS-Protection, X-Content-Type-Options, Referrer-Policy |
| API | Rate limiting | 5000 req/15min per IP via express-rate-limit |
| Auth | Password hashing | bcrypt with salt rounds 10 |
| Auth | JWT tokens | 7-day expiry, server-side secret |
| Network | Security Groups | SSH/app ports restricted to specific CIDRs |
| Network | IMDSv2 enforced | Mitigates SSRF on EC2 metadata |
| Infra | Encrypted EBS | Root volume encrypted at rest |
| CI | Trivy scan | Filesystem vulnerability scanning |
| CI | Gitleaks | Secret detection in git history |
| CI | Snyk | Dependency vulnerability scanning |
| Config | .env file perms | Ansible sets mode 0600 (owner read/write only) |
| Docker | Non-routable default CIDRs | Forces explicit IP allowlisting |

### Known Security Gaps (Be Honest!)
1. **SQL injection in ClickHouse**: `log.controller.js` uses string interpolation: `WHERE deployment_id = '${id}'`
2. **No JWT verification middleware**: Protected routes check Redux state, not backend tokens
3. **Docker socket exposure**: Application container has host-level Docker access
4. **AWS credentials in env vars**: Visible in `docker inspect`
5. **No HTTPS**: No SSL/TLS termination configured
6. **CORS wildcard**: Falls back to `origin: '*'` without `CORS_ORIGIN`

### Counter Questions
1. "How would you fix the ClickHouse SQL injection?"
2. "What's the impact of the Docker socket exposure?"
3. "How would you implement proper JWT middleware?"
4. "What's a CSRF attack and is Vortex vulnerable?"
5. "Can someone spoof another user's GitHub profile?"

---

## Q138-Q150. Security Deep Dive

| # | Question | Key Points |
|---|----------|------------|
| Q138 | "Explain JWT attacks" | Token theft (localStorage), algorithm confusion, no revocation mechanism |
| Q139 | "How would XSS affect Vortex?" | Steal JWT from localStorage; React's JSX escaping helps but isn't complete |
| Q140 | "Is Vortex vulnerable to CSRF?" | Less so — JWT in Authorization header not sent automatically by browser |
| Q141 | "Explain CORS in your setup" | Configurable origin; in prod, should be specific domain |
| Q142 | "Is there SSRF risk?" | Yes — build server clones arbitrary GitHub URLs; could be pointed at internal services |
| Q143 | "NoSQL injection risk?" | Mongoose parameterizes queries, but direct queries to ClickHouse are vulnerable |
| Q144 | "Docker security concerns" | Socket mount = root access; no resource limits on build containers |
| Q145 | "How do you manage secrets?" | Env vars via Ansible; should use Vault or AWS Secrets Manager |
| Q146 | "IAM security" | SSM policy only; EC2 role has minimal permissions |
| Q147 | "How would you add HTTPS?" | Let's Encrypt + certbot + Nginx SSL termination |
| Q148 | "What's the principle of least privilege?" | Applied in security groups (restricted CIDRs) but not in Docker socket access |
| Q149 | "How do you handle dependency vulnerabilities?" | Snyk in CI; `npm audit` for local checks |
| Q150 | "Rate limiting bypass?" | Reverse proxy mode — `trust proxy: 1` ensures X-Forwarded-For used correctly |

---

# SECTION 15 — DEVOPS PRACTICES

---

## Q151-Q165. DevOps Questions

| # | Question | Key Points |
|---|----------|------------|
| Q151 | "Explain your deployment strategy" | Push-to-main triggers CD → Ansible → Docker Compose rebuild → health checks |
| Q152 | "How do you handle rollbacks?" | `rollback.sh`: `git reset --hard HEAD~1` → docker compose down/up |
| Q153 | "What monitoring do you have?" | Health endpoints, Docker health checks, Redpanda Console for Kafka |
| Q154 | "How do you handle logging?" | Application logs via Docker, build logs via Kafka → ClickHouse |
| Q155 | "What's your backup strategy?" | `backup.sh`: mongodump + ClickHouse schema export |
| Q156 | "Blue-green deployment?" | Not implemented; would need two Docker Compose stacks + Nginx upstream swap |
| Q157 | "Canary deployment?" | Not implemented; would need traffic splitting at Nginx level |
| Q158 | "How do you scale?" | Vertically (bigger EC2) or horizontally (multiple instances + load balancer) |
| Q159 | "What's infrastructure as code?" | Terraform for provisioning, Ansible for configuration — reproducible infrastructure |
| Q160 | "GitOps?" | Partial — push to main triggers deployment, but no ArgoCD/Flux-style reconciliation |
| Q161 | "How do you handle config drift?" | Ansible idempotency re-applies desired state on each run |
| Q162 | "What's your incident response?" | Health check failures → CD diagnostics step → manual SSH for deep debugging |
| Q163 | "How do you handle secrets rotation?" | Update GitHub secrets → re-run CD pipeline → Ansible regenerates .env |
| Q164 | "What SLA could Vortex provide?" | Single EC2 = ~99.5% (EC2 SLA); would need multi-AZ for 99.99% |
| Q165 | "How do you handle capacity planning?" | t3.large chosen based on Kafka/ClickHouse memory requirements + swap for builds |

---

# SECTION 16 — LOW LEVEL DESIGN

---

## Q166. "Design the deployment service."

### Ideal Answer
```
┌─────────────────────────────────────────────────┐
│              DeploymentService                   │
├─────────────────────────────────────────────────┤
│ + startDeployment(req: DeployRequest): DeployID │
│ + getDeploymentStatus(id: DeployID): Status      │
│ + getDeploymentLogs(id: DeployID): Log[]         │
│ + cancelDeployment(id: DeployID): void           │
│ + listDeployments(user: string): Deployment[]    │
├─────────────────────────────────────────────────┤
│ - buildQueue: Queue<BuildJob>                    │
│ - containerManager: DockerClient                 │
│ - logPublisher: KafkaProducer                    │
│ - artifactStore: S3Client                        │
│ - metadataStore: MongoDB                         │
└─────────────────────────────────────────────────┘

DeployRequest {
  userId: string
  repoUrl: string
  branch: string
  envVars: Map<string, string>
}

BuildJob {
  deploymentId: UUID
  status: QUEUED | BUILDING | UPLOADING | COMPLETED | FAILED
  startedAt: DateTime
  completedAt: DateTime
  containerName: string
  buildStrategy: string
}
```

**Current Vortex Implementation vs. Ideal:**
| Aspect | Current | Ideal |
|--------|---------|-------|
| Job queue | Direct Docker exec | BullMQ with Redis backend |
| Status tracking | None | State machine in MongoDB |
| Cancellation | Not implemented | `docker stop` + status update |
| Timeout | None | 10-minute build timeout |
| Retries | None | 3 retries with exponential backoff |
| Resource limits | None | Docker --memory, --cpus flags |

---

## Q167-Q172. Low Level Design Questions

| # | Question | Key Points |
|---|----------|------------|
| Q167 | "Design the GitHub webhook handler" | POST webhook → validate signature → parse push event → trigger deployment |
| Q168 | "Design the build queue" | FIFO queue with priority; max concurrent builds; dead letter queue for failures |
| Q169 | "Design the deployment log system" | Producer → Kafka → ClickHouse with TTL; WebSocket streaming to frontend |
| Q170 | "Design the build worker" | Isolated container with resource limits; timeout watchdog; cleanup on exit |
| Q171 | "Design a notification system" | Build status events → notification service → email/Slack/webhook |
| Q172 | "Design the deployment status state machine" | PENDING → QUEUED → CLONING → BUILDING → UPLOADING → DEPLOYED / FAILED |

---

# SECTION 17 — SYSTEM DESIGN / SCALING

---

## Q173. "How would you scale Vortex to 100,000 users?"

### Ideal Answer

```
Scale Milestones:

100 users (current):
├── Single EC2 t3.large
├── Docker Compose
├── All services co-located
└── Bottleneck: Concurrent builds (2-3 max)

1,000 users:
├── Larger EC2 (t3.xlarge / c5.2xlarge)
├── Separate MongoDB to Atlas
├── Add build queue (BullMQ)
├── Limit concurrent builds to 5
└── Bottleneck: Single-host Docker

10,000 users:
├── Kubernetes (EKS)
├── Kafka → Amazon MSK (managed)
├── ClickHouse → ClickHouse Cloud
├── MongoDB → Atlas
├── S3 + CloudFront CDN
├── Multiple build worker pods
├── HPA for auto-scaling workers
└── Bottleneck: Build queue throughput

100,000 users:
├── Multi-region deployment
├── Build workers on Spot instances
├── Global CDN for deployed sites
├── Custom domain support
├── Tenant isolation per namespace
├── Redis for session/cache
├── Rate limiting per user (not just IP)
└── Bottleneck: Cost optimization

1,000,000+ users:
├── Serverless build workers (Firecracker VMs)
├── Global deployment edge network
├── Multi-AZ Kafka with geo-replication
├── Read replicas for MongoDB
├── Sharding by user_id
├── Team/org hierarchy
├── Usage-based billing
└── Bottleneck: Operational complexity
```

### Counter Questions
1. "At what point does Docker Compose stop being viable?"
2. "How would you handle 100 concurrent builds?"
3. "What's the cost at 100K users?"
4. "How do you handle noisy neighbor problems?"
5. "Would you use serverless for any component?"

---

## Q174-Q178. Scaling Questions

| # | Question | Key Points |
|---|----------|------------|
| Q174 | "Database sharding strategy" | Shard by username/user_id for even distribution |
| Q175 | "Caching strategy" | Redis for GitHub API responses, user sessions, deployment status |
| Q176 | "CDN strategy" | CloudFront for deployed static sites; cache S3 objects at edge |
| Q177 | "Build queue optimization" | Priority queues, concurrent limits per user, spot instances for workers |
| Q178 | "Multi-region deployment" | Route 53 latency-based routing, per-region build workers, global ClickHouse |

---

# SECTION 18 — FAILURE SCENARIOS

---

## Q179-Q196. Failure Scenarios

| # | Scenario | What Happens | How to Recover |
|---|----------|-------------|----------------|
| Q179 | MongoDB crashes | API returns 500; health endpoint shows `database: DOWN`; new registrations/logins fail | `unless-stopped` auto-restarts; data survives in named volume |
| Q180 | Kafka crashes | Build logs lost during builds; ClickHouse stops ingesting | Auto-restart; messages after recovery are fine; logs during outage are lost |
| Q181 | ClickHouse crashes | Log queries fail with 500; build logs accumulate in Kafka | Auto-restart; Kafka retains messages; ClickHouse catches up on reconnect |
| Q182 | Docker daemon crashes | ALL containers stop | Systemd restarts Docker; `unless-stopped` restarts containers |
| Q183 | EC2 crashes | Total outage | Elastic IP preserved; launch new instance; Ansible redeploys |
| Q184 | S3 unavailable | Build artifacts can't be uploaded; deployment "succeeds" but site is broken | Build server logs error; manual re-deploy after S3 recovers |
| Q185 | GitHub API fails | Repository/branch listing fails; builds that need clone may fail | Octokit throws 500; frontend shows error; retry manually |
| Q186 | Build server fails | Deployment shows error logs in Kafka; container exits non-zero | Check logs via ClickHouse; fix code; re-deploy |
| Q187 | Deployment stops midway | Orphaned container on host; partial S3 upload | Force-remove container; clean S3 prefix; re-deploy |
| Q188 | Terraform apply fails | Partial infrastructure state | `terraform plan` to assess damage; fix and re-apply; state may need manual edit |
| Q189 | Ansible fails | Partial deployment; some containers may be down | Re-run playbook (idempotent); check which task failed |
| Q190 | Docker image missing | Deploy controller tries to build from `/home/ubuntu/vortex/build-server` | Auto-fallback in deploy controller; fails if source code isn't on host |
| Q191 | Build takes too long | No timeout; container runs indefinitely | Need to implement `docker stop` after configurable timeout |
| Q192 | Huge repository uploaded | Disk fills during git clone or build | Need max repo size limit; 30GB EBS may not suffice |
| Q193 | Container dies mid-build | Logs stop in Kafka; no completion signal | Implement watchdog; `docker wait` in deploy controller catches exit |
| Q194 | Disk fills | Docker can't create containers; writes fail | Monitor with CloudWatch; add disk alerts; increase EBS volume |
| Q195 | RAM fills | OOM killer terminates processes | 4GB swap helps; Kafka heap limited to 256MB; monitor and right-size |
| Q196 | CPU 100% | Build performance degrades; API latency spikes | Docker resource limits (`--cpus`); scale horizontally |

---

# SECTION 19 — TRADEOFFS

---

## Q197-Q215. Tradeoffs Analysis

| # | Decision | Why This | Why Not Alternative | Tradeoff |
|---|----------|----------|-------------------|----------|
| Q197 | Docker Compose | Simple single-host orchestration | K8s: overhead for single host | Simplicity vs. scalability |
| Q198 | KRaft Kafka | No ZooKeeper dependency | ZooKeeper: battle-tested | Simplicity vs. maturity |
| Q199 | ClickHouse | Column-store perfect for logs | Elasticsearch: full-text search | Query speed vs. flexibility |
| Q200 | MongoDB | Document model, rapid dev | PostgreSQL: ACID, relations | Flexibility vs. consistency |
| Q201 | Express | Mature, middleware ecosystem | Fastify: 2x faster | Ecosystem vs. performance |
| Q202 | Vite | Fast HMR, ESM-native | Webpack: more plugins | Speed vs. ecosystem |
| Q203 | Redux Toolkit | Persistence, DevTools | Context API: simpler | Structure vs. simplicity |
| Q204 | Terraform | Declarative IaC, state tracking | CloudFormation: AWS-native | Multi-cloud vs. deep integration |
| Q205 | Ansible | Agentless, SSH-based | Chef/Puppet: agent-based | Simplicity vs. convergence |
| Q206 | GitHub Actions | Integrated with GitHub | Jenkins: more flexible | Integration vs. customization |
| Q207 | EC2 | Full control, persistent services | ECS/Lambda: managed | Control vs. operational overhead |
| Q208 | S3 | Cheap, durable, CDN-compatible | EFS: file system semantics | Simplicity vs. POSIX compliance |
| Q209 | bcrypt | Battle-tested, adequate speed | argon2: more resistant | Compatibility vs. security margin |
| Q210 | JWT | Stateless, no session store | Session cookies: revocable | Scalability vs. revocability |
| Q211 | Polling | Simple implementation | WebSocket: real-time | Simplicity vs. efficiency |
| Q212 | execSync Docker | Simple, synchronous check | Dockerode: async, type-safe | Simplicity vs. robustness |
| Q213 | Single EC2 | Cost-effective | Multi-AZ: high availability | Cost vs. reliability |
| Q214 | ubuntu:focal (build) | Full toolchain available | Alpine: smaller image | Compatibility vs. size |
| Q215 | String interpolation (CH) | Quick implementation | Parameterized queries: secure | Speed vs. security |

---

# SECTION 20 — RESUME DEEP DIVE

---

## Q216-Q225. Resume Questions

| # | Question | What They're Testing |
|---|----------|---------------------|
| Q216 | "Walk me through each technology on your resume" | Can you explain why each was chosen? |
| Q217 | "What metrics can you share?" | Build times, deployment frequency, log throughput |
| Q218 | "How many deployments has Vortex handled?" | Honest answer about scale |
| Q219 | "What's the Kafka throughput?" | Benchmark: 1M records × 100 bytes |
| Q220 | "How many concurrent users supported?" | Honest: 5-10 concurrent builds on t3.large |
| Q221 | "What's the average build time?" | Depends on project size; ~1-3 minutes for typical React apps |
| Q222 | "How did you learn these technologies?" | Documentation, experimentation, debugging |
| Q223 | "What's your strongest technology?" | Node.js/Express backend + Docker orchestration |
| Q224 | "What's your weakest area?" | Kubernetes, advanced Kafka tuning |
| Q225 | "If I gave you a production incident at 3 AM, what would you do?" | SSH → docker logs → health checks → identify failing component → fix/rollback |

---

# SECTION 21 — BEHAVIORAL

---

## Q226-Q240. Behavioral Questions

| # | Question | Framework | Key Points |
|---|----------|-----------|------------|
| Q226 | "Tell me about a conflict" | STAR | Debugging Docker-in-Docker; disagreement on approach; resolved by testing both |
| Q227 | "Tell me about a failure" | STAR | Initial design without Kafka — direct ClickHouse writes caused data loss; redesigned |
| Q228 | "How do you handle pressure?" | STAR | Deployment deadline; prioritized core features; shipped MVP, iterated |
| Q229 | "Describe a learning moment" | STAR | Learning Terraform state management after accidentally corrupting state |
| Q230 | "How do you take ownership?" | STAR | Sole developer; owned every decision, every bug, every deployment |
| Q231 | "Debugging story" | STAR | Kafka consumer lag causing delayed logs; discovered heap size too small; fixed |
| Q232 | "Leadership example" | STAR | Documented architecture decisions for future contributors |
| Q233 | "When did you go above and beyond?" | STAR | Added CI security scanning (Trivy, Gitleaks, Snyk) beyond basic requirements |
| Q234 | "How do you prioritize?" | Framework | Must-have (deploy pipeline) → Should-have (monitoring) → Nice-to-have (custom domains) |
| Q235 | "Describe a difficult technical decision" | Tradeoff | Choosing Kafka over Redis Streams; complexity vs. durability |
| Q236 | "How do you handle ambiguity?" | Process | Research → prototype → validate → implement |
| Q237 | "What's your development process?" | Workflow | Design → implement → test → deploy → monitor |
| Q238 | "How do you stay current?" | Learning | Documentation, open source, hands-on projects |
| Q239 | "Why should we hire you?" | Value | Full-stack systems thinking; can build and operate end-to-end |
| Q240 | "Where do you see yourself in 5 years?" | Growth | Staff/Principal engineer level; designing systems at scale |

---

# SECTION 22 — CODING CHALLENGES

---

## Q241. "Write a JWT verification middleware."

```javascript
// What you SHOULD have in your codebase (but don't):
import jwt from 'jsonwebtoken';

export const verifyToken = (req, res, next) => {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
        return res.status(401).json({ message: 'No token provided' });
    }

    const token = authHeader.split(' ')[1];
    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.userId = decoded.id;
        next();
    } catch (error) {
        if (error.name === 'TokenExpiredError') {
            return res.status(401).json({ message: 'Token expired' });
        }
        return res.status(403).json({ message: 'Invalid token' });
    }
};
```

## Q242. "Write a retry wrapper for async operations."

```javascript
async function withRetry(fn, maxRetries = 3, delay = 1000) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            return await fn();
        } catch (error) {
            if (attempt === maxRetries) throw error;
            console.log(`Attempt ${attempt} failed. Retrying in ${delay}ms...`);
            await new Promise(resolve => setTimeout(resolve, delay * attempt));
        }
    }
}

// Usage:
await withRetry(() => producer.send({ topic: 'build-logs', messages: [...] }));
```

## Q243. "Write a deployment status state machine."

```javascript
const STATES = {
    PENDING: 'PENDING',
    QUEUED: 'QUEUED',
    CLONING: 'CLONING',
    BUILDING: 'BUILDING',
    UPLOADING: 'UPLOADING',
    DEPLOYED: 'DEPLOYED',
    FAILED: 'FAILED'
};

const TRANSITIONS = {
    [STATES.PENDING]: [STATES.QUEUED, STATES.FAILED],
    [STATES.QUEUED]: [STATES.CLONING, STATES.FAILED],
    [STATES.CLONING]: [STATES.BUILDING, STATES.FAILED],
    [STATES.BUILDING]: [STATES.UPLOADING, STATES.FAILED],
    [STATES.UPLOADING]: [STATES.DEPLOYED, STATES.FAILED],
    [STATES.DEPLOYED]: [],
    [STATES.FAILED]: [STATES.PENDING] // retry
};

function transition(currentState, newState) {
    if (!TRANSITIONS[currentState]?.includes(newState)) {
        throw new Error(`Invalid transition: ${currentState} → ${newState}`);
    }
    return newState;
}
```

## Q244. "Fix the SQL injection in your ClickHouse query."

```javascript
// BEFORE (vulnerable):
const query = `SELECT ... WHERE deployment_id = '${id}'`;

// AFTER (parameterized):
const result = await client.query({
    query: `SELECT log_uuid, deployment_id, log_message, log_level, 
            created_at AS timestamp
            FROM build_logs
            WHERE deployment_id = {id:String}
            ORDER BY created_at ASC`,
    query_params: { id },
    format: 'JSONEachRow',
});
```

## Q245-Q255. Additional Coding Questions

| # | Question | Key Concept |
|---|----------|------------|
| Q245 | "Write a Docker health check in Node.js" | HTTP server on separate port returning process stats |
| Q246 | "Write a rate limiter from scratch" | Sliding window counter with Map/Redis |
| Q247 | "Write a build strategy detector" | File existence checks + package.json parsing |
| Q248 | "Write a batch S3 uploader" | Promise.allSettled with concurrency limit |
| Q249 | "Write a Terraform output parser" | JSON parsing of `terraform output -json` |
| Q250 | "Write a graceful shutdown handler" | SIGTERM/SIGINT → drain connections → close DB → exit |
| Q251 | "Write a log aggregation query" | ClickHouse GROUP BY deployment_id with COUNT and time ranges |
| Q252 | "Write a Docker container cleanup script" | Remove exited containers, dangling images, unused volumes |
| Q253 | "Write a health check with retry" | Exponential backoff with max retries |
| Q254 | "Write a Kafka consumer group rebalance handler" | kafkajs consumer with `run()` and error handling |
| Q255 | "Write a MongoDB connection with retry" | Mongoose connect with exponential backoff |

---

# SECTION 23 — RAPID FIRE (100 QUESTIONS)

---

> **Instructions:** Answer each in 1-2 sentences max. These test breadth.

| # | Question | Expected Answer |
|---|----------|----------------|
| 1 | What is Docker? | Containerization platform that packages applications with their dependencies. |
| 2 | Docker vs VM? | Docker shares host kernel (lightweight); VMs have full OS (heavy). |
| 3 | What is a Dockerfile? | Instructions to build a Docker image, layer by layer. |
| 4 | What is Docker Compose? | Tool for defining and running multi-container Docker applications. |
| 5 | What is Kubernetes? | Container orchestration platform for managing containerized workloads at scale. |
| 6 | Why not K8s for Vortex? | Single-host deployment; Docker Compose is simpler. |
| 7 | What is Kafka? | Distributed event streaming platform for high-throughput, durable message handling. |
| 8 | Kafka vs RabbitMQ? | Kafka: log-based, replay; RabbitMQ: queue-based, no replay. |
| 9 | What is a Kafka topic? | Named channel for publishing and subscribing to messages. |
| 10 | What is a partition? | Topic subdivision enabling parallel processing and ordering within partition. |
| 11 | What is KRaft? | Kafka's built-in consensus protocol replacing ZooKeeper for metadata management. |
| 12 | What is ClickHouse? | Columnar OLAP database optimized for analytical queries over large datasets. |
| 13 | ClickHouse vs PostgreSQL? | ClickHouse: 100x faster analytics; PostgreSQL: better for OLTP. |
| 14 | What is MergeTree? | ClickHouse's primary storage engine optimized for sorted insert and merge operations. |
| 15 | What is a materialized view? | Pre-computed query result that auto-updates when source data changes. |
| 16 | What is MongoDB? | Document-oriented NoSQL database storing JSON-like BSON documents. |
| 17 | MongoDB vs PostgreSQL? | MongoDB: schema flexibility; PostgreSQL: ACID compliance and relations. |
| 18 | What is Mongoose? | ODM (Object Document Mapper) for MongoDB in Node.js. |
| 19 | What is Express? | Minimal Node.js web framework for building APIs and web apps. |
| 20 | Express vs Fastify? | Express: mature ecosystem; Fastify: 2x performance, schema validation. |
| 21 | What is JWT? | JSON Web Token — stateless authentication via signed JSON payload. |
| 22 | JWT vs Session? | JWT: stateless, scalable; Session: server-stored, revocable. |
| 23 | What is bcrypt? | Password hashing function with adaptive cost factor (salt rounds). |
| 24 | What is Helmet? | Express middleware that sets security-related HTTP headers. |
| 25 | What is CORS? | Cross-Origin Resource Sharing — browser mechanism to allow/restrict cross-domain requests. |
| 26 | What is rate limiting? | Throttling requests per client to prevent abuse/DDoS. |
| 27 | What is Nginx? | High-performance web server and reverse proxy. |
| 28 | Reverse proxy vs forward proxy? | Reverse: protects servers (Nginx); Forward: protects clients (VPN). |
| 29 | What is Terraform? | Infrastructure-as-code tool for declarative cloud resource provisioning. |
| 30 | Terraform vs CloudFormation? | Terraform: multi-cloud; CloudFormation: AWS-only, deeper integration. |
| 31 | What is Terraform state? | JSON file tracking the mapping between config and real infrastructure. |
| 32 | What is Ansible? | Agentless configuration management tool using SSH and YAML playbooks. |
| 33 | Ansible vs Chef? | Ansible: agentless, push-based; Chef: agent-based, pull-based. |
| 34 | What is idempotency? | Running an operation multiple times produces the same result. |
| 35 | What is Jinja2? | Python-based template engine used by Ansible for dynamic config generation. |
| 36 | What is GitHub Actions? | CI/CD platform integrated with GitHub for automated workflows. |
| 37 | CI vs CD? | CI: automated testing/validation; CD: automated deployment. |
| 38 | What is a VPC? | Virtual Private Cloud — isolated network within AWS. |
| 39 | What is a subnet? | Subdivision of a VPC with its own CIDR block and route table. |
| 40 | What is an Internet Gateway? | AWS component that enables VPC resources to communicate with the internet. |
| 41 | What is a security group? | Stateful firewall for EC2 instances controlling inbound/outbound traffic. |
| 42 | Security group vs NACL? | SG: stateful, instance-level; NACL: stateless, subnet-level. |
| 43 | What is an Elastic IP? | Static public IPv4 address that persists across EC2 stop/start cycles. |
| 44 | What is IAM? | Identity and Access Management — AWS service for access control. |
| 45 | What is an IAM role? | AWS identity with permissions that can be assumed by services. |
| 46 | What is S3? | Simple Storage Service — object storage with 99.999999999% durability. |
| 47 | What is EC2? | Elastic Compute Cloud — virtual servers in AWS. |
| 48 | t3.large specs? | 2 vCPUs, 8 GB RAM, up to 5 Gbps network, burstable performance. |
| 49 | What is React? | JavaScript library for building user interfaces with a component-based model. |
| 50 | What is Vite? | Modern frontend build tool with native ESM dev server and Rollup-based production builds. |
| 51 | Vite vs Webpack? | Vite: instant HMR via ESM; Webpack: more mature plugin ecosystem. |
| 52 | What is Redux? | Predictable state container for JavaScript apps with a single source of truth. |
| 53 | Redux vs Context? | Redux: DevTools, middleware, persistence; Context: simpler, no external deps. |
| 54 | What is redux-persist? | Library to save Redux state to localStorage/sessionStorage and rehydrate on load. |
| 55 | What is React Router? | Declarative routing library for React SPAs. |
| 56 | What is Axios? | HTTP client for browser/Node.js with interceptors and request cancellation. |
| 57 | Axios vs fetch? | Axios: auto JSON parsing, interceptors; fetch: native, no deps. |
| 58 | What is gzip? | Compression algorithm reducing HTTP response sizes by 60-80%. |
| 59 | What is sendfile? | Kernel-level file transfer bypassing userspace for better Nginx performance. |
| 60 | What is tcp_nopush? | Sends HTTP headers and first chunk together, reducing round trips. |
| 61 | What is a bridge network? | Docker's default network driver enabling container-to-container communication. |
| 62 | Docker volume vs bind mount? | Volume: managed by Docker; Bind mount: maps host directory directly. |
| 63 | What is ENTRYPOINT? | Docker instruction defining the container's main executable. |
| 64 | ENTRYPOINT vs CMD? | ENTRYPOINT: always runs; CMD: default arguments, overridable. |
| 65 | What is docker.sock? | Unix socket file for communicating with the Docker daemon. |
| 66 | What is IMDSv2? | Instance Metadata Service v2 with session-based token authentication. |
| 67 | What is a consumer group? | Set of Kafka consumers that cooperatively read from topic partitions. |
| 68 | What is an offset? | Sequential position of a message within a Kafka partition. |
| 69 | What is ISR? | In-Sync Replicas — set of replicas fully caught up with the leader. |
| 70 | What is replication factor? | Number of copies of each partition across Kafka brokers. |
| 71 | What is Octokit? | GitHub's official REST API client library for JavaScript. |
| 72 | What is npm ci? | Clean install from lockfile only; faster and more deterministic than npm install. |
| 73 | What is nodemon? | Node.js development tool that auto-restarts on file changes. |
| 74 | What is ESM? | ECMAScript Modules — native JavaScript module system with import/export. |
| 75 | What is graceful shutdown? | Handling termination signals to close connections cleanly before exiting. |
| 76 | What is X-Forwarded-For? | HTTP header identifying the originating IP when behind a proxy. |
| 77 | What is SIGTERM? | Termination signal sent to gracefully stop a process. |
| 78 | SIGTERM vs SIGKILL? | SIGTERM: can be caught and handled; SIGKILL: immediate, unblockable. |
| 79 | What is a health check? | Endpoint that reports service availability and dependency status. |
| 80 | What is Trivy? | Open-source vulnerability scanner for containers and filesystems. |
| 81 | What is Gitleaks? | Tool for detecting secrets and keys committed to git repositories. |
| 82 | What is Snyk? | Developer security platform for finding and fixing vulnerabilities. |
| 83 | What is SonarQube? | Code quality and security analysis platform (`.qodana.yaml` present). |
| 84 | What is NoSQL injection? | Injecting query operators into NoSQL queries to bypass authentication. |
| 85 | What is SSRF? | Server-Side Request Forgery — tricking server into making unintended requests. |
| 86 | What is XSS? | Cross-Site Scripting — injecting malicious scripts into web pages. |
| 87 | What is terraform plan? | Preview of changes Terraform will make without applying them. |
| 88 | What is terraform apply? | Executes the planned changes to create/modify/destroy infrastructure. |
| 89 | What is a Terraform provider? | Plugin that interfaces with a specific cloud or service API. |
| 90 | What is HCL? | HashiCorp Configuration Language — Terraform's declarative syntax. |
| 91 | What is an Ansible role? | Reusable unit of automation with tasks, handlers, templates, and variables. |
| 92 | What is an Ansible handler? | Task that runs only when notified by a changed task. |
| 93 | What is ssh-keyscan? | Tool to collect SSH public keys from servers for known_hosts. |
| 94 | What is npm-run-all? | CLI tool to run multiple npm scripts in parallel or sequentially. |
| 95 | What is mime-types? | Library mapping file extensions to MIME content types. |
| 96 | What is child_process.exec? | Node.js function to spawn a shell and execute a command. |
| 97 | exec vs execSync? | exec: async, non-blocking; execSync: synchronous, blocks event loop. |
| 98 | What is a UUID? | Universally Unique Identifier — 128-bit number for unique identification. |
| 99 | What is blue-green deployment? | Running two identical environments; switching traffic between them. |
| 100 | What is canary deployment? | Gradually routing a percentage of traffic to a new version. |

---

# SECTION 24 — COUNTER-QUESTION DRILL TREES

---

> These simulate an interviewer who keeps digging deeper and deeper. Practice following each tree to its maximum depth.

## Drill Tree 1: Docker

```
Q: "Why Docker?"
    └─ Q: "Why not Podman?"
        └─ A: "Podman is daemonless and rootless. For Vortex, Docker socket mounting 
              is a key pattern. Podman would require different architecture."
    └─ Q: "Why not Kubernetes?"
        └─ A: "Single-host deployment. K8s adds control plane overhead."
            └─ Q: "At what scale would you switch to K8s?"
                └─ A: "When I need multi-node, auto-scaling, or >10 concurrent builds."
    └─ Q: "Why not ECS?"
        └─ A: "ECS ties you to AWS. Docker Compose is cloud-agnostic."
            └─ Q: "But you're already on AWS with Terraform."
                └─ A: "True, but the app layer is portable. Only infra is AWS-specific."
    └─ Q: "Docker daemon crashes. What happens?"
        └─ A: "All containers stop. systemd restarts Docker. unless-stopped restarts containers."
            └─ Q: "What about in-flight builds?"
                └─ A: "Lost. Need persistent build state in MongoDB/Redis for recovery."
                    └─ Q: "How would you design that?"
                        └─ A: "Checkpoint build stages in Redis. On restart, 
                              resume from last checkpoint."
```

## Drill Tree 2: Kafka

```
Q: "Why Kafka for logs?"
    └─ Q: "Why not Redis Streams?"
        └─ A: "Kafka provides stronger durability guarantees, native ClickHouse integration, 
              and multi-consumer support."
            └─ Q: "But Redis is simpler."
                └─ A: "True. For < 100 concurrent builds, Redis Streams would suffice. 
                      I chose Kafka to learn distributed streaming and for the 
                      ClickHouse Kafka Engine integration."
    └─ Q: "Why not write directly to ClickHouse?"
        └─ A: "Direct writes couple the build server to ClickHouse. If CH is down, 
              build logs are lost. Kafka provides a durable buffer."
            └─ Q: "But with RF=1, Kafka isn't that durable either."
                └─ A: "Correct. In production, I'd increase to RF=3 for the build-logs topic. 
                      The current RF=1 is for development simplicity."
    └─ Q: "Your producer awaits every message. Performance?"
        └─ A: "Each await adds ~5ms latency. For 1000 log lines, that's 5 seconds overhead. 
              I'd batch with linger.ms and batch.size for production."
            └─ Q: "What's exactly-once semantics and do you need it?"
                └─ A: "EOS ensures no duplicates even with retries. For logs, 
                      at-least-once with dedup via log_uuid is sufficient."
```

## Drill Tree 3: Security

```
Q: "Is your app secure?"
    └─ Q: "I see string interpolation in ClickHouse queries."
        └─ A: "Yes, that's a SQL injection vulnerability. I should use parameterized queries 
              with the ClickHouse client's query_params feature."
            └─ Q: "What could an attacker do?"
                └─ A: "Inject ClickHouse SQL to read other users' logs, 
                      drop tables, or extract system info."
    └─ Q: "Docker socket is mounted. Impact?"
        └─ A: "Full host access. An attacker could create privileged containers, 
              access host filesystem, or install backdoors."
            └─ Q: "How would you mitigate?"
                └─ A: "1. Rootless Docker mode. 2. Docker socket proxy with 
                      read-only filter. 3. Separate build worker process on host. 
                      4. gVisor/Firecracker for sandboxing."
    └─ Q: "No JWT middleware on API routes?"
        └─ A: "Currently, protected routes are only guarded on the frontend. 
              API endpoints are technically accessible without auth."
            └─ Q: "So anyone can trigger a deployment?"
                └─ A: "Yes, if they know the API endpoint. Critical fix needed: 
                      JWT middleware on deploy, user, and log routes."
```

## Drill Tree 4: Scaling

```
Q: "Can this handle 1000 concurrent builds?"
    └─ A: "No. Single EC2, no build queue, no resource limits."
        └─ Q: "How would you redesign it?"
            └─ A: "BullMQ queue → Kubernetes build workers with HPA → 
                  horizontal pod autoscaling based on queue depth."
                └─ Q: "What's the bottleneck first?"
                    └─ A: "Disk I/O during git clone and npm install. 
                          Each build needs ~500MB-2GB temporary space."
                        └─ Q: "How do you solve that?"
                            └─ A: "1. Build caching (npm/yarn cache volumes). 
                                  2. Git shallow clones (--depth 1). 
                                  3. Ephemeral NVMe instances for builds."
```

## Drill Tree 5: Database

```
Q: "Why MongoDB over PostgreSQL?"
    └─ Q: "What if you need transactions?"
        └─ A: "MongoDB 4.0+ supports multi-document transactions. 
              But current workload is single-document operations only."
    └─ Q: "Your user schema has no indexes beyond unique fields."
        └─ A: "Correct. For the current scale, full scans are fast enough. 
              For production: compound index on { username: 1, email: 1 }, 
              and { username: 1 } on deployments."
            └─ Q: "What's the explain plan for getDeploymentsByUser?"
                └─ A: "Without an index on username, it's a COLLSCAN. 
                      With index, it's an IXSCAN with covered query potential."
    └─ Q: "How would you migrate to PostgreSQL?"
        └─ A: "1. Design normalized schema (users, deployments, logs tables). 
              2. Write migration scripts with data transformation. 
              3. Use Prisma or TypeORM for the ORM layer. 
              4. Run dual-write during migration, then cut over."
```

---

# APPENDIX A — INTERVIEW TIPS

---

## General Rules

1. **Always start with the "what" then the "why"** — Don't just describe, explain the reasoning
2. **Be honest about gaps** — Interviewers respect "I know this is a security gap, here's how I'd fix it" far more than trying to hide it
3. **Use your actual code as evidence** — "In line 17 of deploy.controller.js, I do X because Y"
4. **Quantify when possible** — "3-node Kafka cluster with 256MB heap each" not "a Kafka cluster"
5. **Show tradeoff awareness** — Every decision has pros and cons; show you considered both
6. **Have the counter-question ready** — When they ask "why X", preemptively address "why not Y"
7. **Don't memorize — understand** — If you understand the architecture, you can derive any answer

## Things That ALWAYS Impress
- Mentioning the ClickHouse Kafka Engine → Materialized View → MergeTree pipeline
- Explaining Docker-in-Docker security implications honestly
- Knowing your Terraform state file shouldn't be committed
- Understanding that `execSync` blocks the event loop
- Acknowledging the SQL injection in `log.controller.js`
- Explaining KRaft vs. ZooKeeper trade-offs

## Things That ALWAYS Fail
- "I used Kafka because it's popular"
- "MongoDB is better than PostgreSQL"
- "Docker Compose is production-ready for any scale"
- "My app is secure" (without nuance)
- "I followed a tutorial" (even if you did, frame it as learning + customization)

---

# APPENDIX B — QUICK REFERENCE CARD

---

## File Locations
| Component | Path |
|-----------|------|
| Backend Server | `backend/server.js` |
| Auth Controller | `backend/controllers/auth.controller.js` |
| Deploy Controller | `backend/controllers/deploy.controller.js` |
| Log Controller | `backend/controllers/log.controller.js` |
| Build Server Script | `build-server/script.js` |
| Build Server Shell | `build-server/main.sh` |
| Docker Compose | `services/docker-compose.yml` |
| ClickHouse Init | `services/init-table.sql` |
| Nginx Config | `nginx/nginx.conf` |
| Terraform EC2 | `infra/terraform/ec2.tf` |
| Terraform Security | `infra/terraform/security.tf` |
| Ansible Playbook | `infra/ansible/playbook.yml` |
| CI Pipeline | `.github/workflows/ci.yml` |
| CD Pipeline | `.github/workflows/cd.yml` |
| Redux Store | `frontend/src/redux/store.js` |
| App Routes | `frontend/src/App.jsx` |

## Key Numbers
| Metric | Value |
|--------|-------|
| Kafka brokers | 3 (KRaft mode) |
| Kafka heap | 256MB max / 128MB init per broker |
| Rate limit | 5000 req / 15 min |
| JWT expiry | 7 days |
| Auto-logout | 24 hours |
| bcrypt salt rounds | 10 |
| EBS volume | 30GB gp3 |
| Swap space | 4GB |
| S3 upload batch | 5 concurrent files |
| Health check retries (CD) | 12 × 15s = 3 min |
| Health check retries (Ansible) | 12 × 10s = 2 min |
| VPC CIDR | 10.0.0.0/16 |
| Subnet CIDR | 10.0.1.0/24 |
| EC2 type | t3.large (2 vCPU, 8GB RAM) |
| Security group rules | 21 (ingress + egress) |

---

> **Total Questions: 355 (Q1-Q255 + 100 Rapid Fire)**
>
> Study this handbook section by section. For each question, practice speaking your answer out loud for 2-3 minutes. The counter-question drill trees in Section 24 are your most important practice material — they simulate what a real interview feels like.
>
> **You built Vortex. Own every line of code. Own every decision. Own every tradeoff.**
