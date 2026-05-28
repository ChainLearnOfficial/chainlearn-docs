# Architecture Overview

ChainLearn is a distributed system composed of seven repositories, each owning a distinct domain boundary. This document describes the system topology, communication patterns, and technology choices.

## System Diagram

```
                         +-----------------+
                         |    Learner's    |
                         |     Browser     |
                         +--------+--------+
                                  |
                          HTTPS / WebSocket
                                  |
                         +--------v--------+
                         |  chainlearn-    |
                         |   frontend      |
                         |  (React + TS)   |
                         +--------+--------+
                                  |
                              REST API
                                  |
              +-------------------+-------------------+
              |                                       |
     +--------v--------+                    +---------v---------+
     |  chainlearn-api  |                    |   chainlearn-ai   |
     |  (Node/Express)  |--- gRPC / HTTP --+|   (Python/FastAPI)|
     +--------+--------+                    +-------------------+
              |
              | SQL                 Soroban RPC
              |                          |
     +--------v--------+       +---------v---------+
     |    PostgreSQL    |       | Stellar Network   |
     |    (primary DB)  |       |  (Soroban VM)     |
     +-----------------+       +--------+----------+
                                         |
                                         | Event Stream
                                +--------v---------+
                                | chainlearn-indexer|
                                |   (Node.js)      |
                                +--------+---------+
                                         |
                                         | REST API
                                         |
                                +--------v---------+
                                | chainlearn-api   |
                                | (query layer)    |
                                +------------------+
```

## Repository Structure

### 1. `chainlearn-frontend`

**Purpose:** Single-page application serving the learner and admin UIs.

**Stack:** React 18, TypeScript, Vite, TanStack Query, Tailwind CSS.

**Responsibilities:**
- User authentication (delegates to `chainlearn-api`)
- Course catalog browsing and enrollment
- Quiz taking and progress viewing
- Wallet connection (Freighter/Albedo) for on-chain rewards
- Credential gallery and sharing

**Why separate:** Frontend iterates on UI/UX independently of backend APIs. Separate repo enables distinct CI pipelines, faster builds, and independent deployment to CDN.

### 2. `chainlearn-api`

**Purpose:** Central REST API handling authentication, business logic, and data persistence.

**Stack:** Node.js 20, Express, Prisma ORM, PostgreSQL, Redis.

**Responsibilities:**
- JWT-based authentication with Stellar SEP-10 challenge/response
- User profile management
- Course and module CRUD
- Quiz submission and scoring orchestration
- Reward distribution coordination (calls smart contracts)
- Credential issuance coordination
- Indexer data aggregation for queries

**Why separate:** This is the primary backend service. It owns the database schema, enforces business rules, and coordinates between the AI service, blockchain, and frontend. Separation from the frontend allows independent scaling and deployment.

### 3. `chainlearn-ai`

**Purpose:** AI microservice for content generation using Cohere's language models.

**Stack:** Python 3.12, FastAPI, Cohere SDK, Redis (caching).

**Responsibilities:**
- Course content generation from topic descriptions
- Quiz question generation with difficulty calibration
- Answer evaluation and feedback generation
- Embedding-based content similarity search

**Why separate:** AI workloads have different scaling characteristics (GPU-bound, bursty, higher memory). Python is the natural language for ML tooling. Isolating this service prevents AI load from affecting API latency.

### 4. `chainlearn-contracts`

**Purpose:** Soroban smart contracts deployed to the Stellar network.

**Stack:** Rust, Soroban SDK, SEP-41 token standard.

**Contracts:**
- **LearnToken** -- SEP-41 fungible token for learner rewards
- **CredentialNFT** -- Non-transferable NFT for course completion credentials
- **ProgressTracker** -- On-chain enrollment and progress state

**Why separate:** Smart contracts require specialized tooling (Soroban CLI, WASM compilation), rigorous testing, and formal audit processes. They have a fundamentally different deployment lifecycle (immutable once deployed).

### 5. `chainlearn-indexer`

**Purpose:** Reads Stellar/Soroban event streams and indexes on-chain data into PostgreSQL for fast querying.

**Stack:** Node.js 20, Stellar SDK, PostgreSQL.

**Responsibilities:**
- Subscribe to Soroban contract events
- Parse and transform on-chain data
- Write indexed records to PostgreSQL
- Provide historical query capabilities

**Why separate:** Indexing is a specialized, long-running process with distinct failure modes. Separating it prevents indexer bugs from affecting the API and allows independent restarts without downtime.

### 6. `chainlearn-docs`

**Purpose:** This repository. Centralized documentation for the entire platform.

**Why separate:** Documentation is a first-class artifact. A dedicated repo enables documentation reviews, version tracking, and publishing to a docs site without coupling to any service's release cycle.

### 7. `chainlearn-infra`

**Purpose:** Infrastructure-as-code for all environments.

**Stack:** Terraform, Docker Compose, GitHub Actions.

**Responsibilities:**
- Docker Compose for local development
- Terraform modules for AWS/GCP provisioning
- CI/CD pipeline definitions
- Environment configuration management

**Why separate:** Infrastructure changes carry high risk and require distinct review processes. Separating infra from application code prevents accidental infrastructure changes in service PRs.

## Communication Patterns

| From | To | Protocol | Purpose |
|------|----|----------|---------|
| Frontend | API | HTTPS (REST) | All user-facing operations |
| Frontend | Stellar | Soroban RPC | Wallet signing (via Freighter) |
| API | AI Service | HTTP | Content generation requests |
| API | Stellar | Soroban RPC | Contract invocations (rewards, credentials) |
| API | PostgreSQL | SQL | Primary data storage |
| API | Redis | Redis protocol | Session cache, rate limiting |
| Indexer | Stellar | Event stream | On-chain event subscription |
| Indexer | PostgreSQL | SQL | Indexed data storage |
| AI Service | Cohere | HTTPS | LLM inference |
| AI Service | Redis | Redis protocol | Generation cache |

## Tech Stack Summary

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend | React + TypeScript | 18.x / 5.x |
| Build tool | Vite | 5.x |
| Styling | Tailwind CSS | 3.x |
| API server | Express (Node.js) | 4.x / 20.x |
| ORM | Prisma | 5.x |
| Database | PostgreSQL | 16.x |
| Cache | Redis | 7.x |
| AI service | FastAPI (Python) | 0.110+ |
| AI provider | Cohere | SDK v5 |
| Smart contracts | Rust + Soroban | 2024.x |
| Blockchain | Stellar (testnet/mainnet) | v21 |
| Indexer | Node.js + Stellar SDK | 20.x / 12.x |
| Infra | Terraform + Docker | 1.7+ / 25.x |
| CI/CD | GitHub Actions | -- |

## Design Principles

1. **Separation of concerns** -- Each repo owns one domain boundary. Services communicate via well-defined APIs, not shared databases.
2. **Blockchain as settlement layer** -- On-chain state is authoritative for rewards and credentials. Off-chain databases are caches and operational stores.
3. **AI as a service** -- Content generation is a dependency, not a core loop. The platform works with pre-authored content if the AI service is unavailable.
4. **Progressive decentralization** -- Start with a centralized API for speed, migrate trust-sensitive operations to smart contracts over time.
5. **Stellar-native patterns** -- Use SEP standards (SEP-10 for auth, SEP-41 for tokens) instead of reinventing primitives.
