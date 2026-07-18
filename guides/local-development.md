# Local Development Guide

This guide walks through setting up the complete ChainLearn platform for local development. You will run all services locally using Docker Compose and connect to the Stellar testnet for blockchain operations.

## Prerequisites

- Docker and Docker Compose (v2+)
- Node.js 20.x and npm 10+
- Python 3.12+
- Rust toolchain with `wasm32-unknown-unknown` target
- [Soroban CLI](https://soroban.stellar.org/docs/getting-started/setup)
- Git

## Architecture Overview

ChainLearn is a multi-repo project with 7 independent packages:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Frontend    │────▶│   API Server │────▶│  AI Service   │
│   (Next.js)  │◀────│   (Fastify)  │◀────│  (FastAPI)    │
└──────┬───────┘     └──────┬───────┘     └───────────────┘
       │                    │
       ▼                    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Stellar    │     │  PostgreSQL  │     │   Indexer    │
│   Network    │     │  + Redis     │     │  (Fastify)   │
│  (Soroban)   │     └──────────────┘     └──────┬───────┘
└──────────────┘                                 ▼
                                         ┌──────────────┐
                                         │   Stellar    │
                                         │   Network    │
                                         └──────────────┘
```

| Package | Stack | Port |
|---|---|---|
| chainlearn-api | Node, Fastify, TypeScript, Drizzle ORM | 3000 |
| chainlearn-frontend | Next.js 14, Tailwind, shadcn/ui | 3000 |
| chainlearn-ai | Python, FastAPI, Cohere | 8000 |
| chainlearn-indexer | Node, Fastify, TypeScript | 3100 |
| chainlearn-contracts | Rust, Soroban SDK | N/A |
| chainlearn-infra | Terraform, Docker Compose | N/A |
| chainlearn-docs | Markdown, OpenAPI | N/A |

## Repository Setup

Clone all repositories into a workspace directory:

```bash
mkdir chainlearn && cd chainlearn

git clone git@github.com:chainlearn/chainlearn-frontend.git
git clone git@github.com:chainlearn/chainlearn-api.git
git clone git@github.com:chainlearn/chainlearn-ai.git
git clone git@github.com:chainlearn/chainlearn-contracts.git
git clone git@github.com:chainlearn/chainlearn-indexer.git
git clone git@github.com:chainlearn/chainlearn-infra.git
git clone git@github.com:chainlearn/chainlearn-docs.git
```

## Quick Start with Docker Compose

The fastest way to run the full stack:

```bash
cd chainlearn-infra
cp .env.example .env
docker compose up -d
```

This starts:
- PostgreSQL (port 5432)
- Redis (port 6379)
- chainlearn-api (port 3000)
- chainlearn-ai (port 8000)
- chainlearn-indexer (port 3001)
- chainlearn-frontend (port 5173)

Access the app at `http://localhost:5173`.

## Manual Setup (Individual Services)

### 1. Database

Start PostgreSQL and Redis:

```bash
cd chainlearn-infra
docker compose up -d postgres redis
```

Or install locally:
- PostgreSQL 16 with database `chainlearn_dev`
- Redis 7

### 2. chainlearn-api

```bash
cd chainlearn-api

# Install dependencies
npm install

# Set up environment
cp .env.example .env
# Edit .env with your database URL and Stellar keys

# Run database migrations (uses Drizzle ORM)
npm run db:generate   # Generate migration SQL
npm run db:migrate    # Run migrations

# Seed the database with sample courses
npm run db:seed

# Start the server
npm run dev
```

The API runs at `http://localhost:3000`.

**Required environment variables:**

```env
DATABASE_URL=postgresql://chainlearn:chainlearn@localhost:5432/chainlearn_dev
REDIS_URL=redis://localhost:6379
JWT_SECRET=dev-secret-change-in-production
STELLAR_NETWORK=testnet
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
LEARN_TOKEN_ADDRESS=...  # From contract deployment
CREDENTIAL_NFT_ADDRESS=...
PROGRESS_TRACKER_ADDRESS=...
ADMIN_SECRET_KEY=...     # Testnet signing key
AI_SERVICE_URL=http://localhost:8000
```

### 3. chainlearn-ai

```bash
cd chainlearn-ai

# Create virtual environment
python -m venv venv
source venv/bin/activate  # or: venv\Scripts\activate on Windows

# Install dependencies
pip install -r requirements.txt

# Set up environment
cp .env.example .env
# Add your Cohere API key

# Start the server
uvicorn src.main:app --reload --port 8000
```

**Required environment variables:**

```env
COHERE_API_KEY=your-cohere-api-key
REDIS_URL=redis://localhost:6379
```

### 4. chainlearn-contracts

```bash
cd chainlearn-contracts

# Install the WASM target
rustup target add wasm32-unknown-unknown

# Build contracts
cargo build --release --target wasm32-unknown-unknown

# Run tests
cargo test

# Deploy to testnet (see contracts/deployment.md for full guide)
soroban keys generate deployer --network testnet
curl "https://friendbot.stellar.org?addr=$(soroban keys address deployer)"

# Deploy each contract
./scripts/deploy.sh testnet
```

### 5. chainlearn-indexer

```bash
cd chainlearn-indexer

npm install

cp .env.example .env
# Configure database URL and contract addresses

npm run migrate   # Run database migrations
npm run dev
```

**Required environment variables:**

```env
DATABASE_URL=postgresql://chainlearn:chainlearn@localhost:5432/chainlearn_dev
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
LEARN_TOKEN_CONTRACT=...
CREDENTIAL_NFT_CONTRACT=...
PROGRESS_TRACKER_CONTRACT=...
POLL_INTERVAL_MS=5000
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
```

### 6. chainlearn-frontend

```bash
cd chainlearn-frontend

npm install

cp .env.example .env.local
# Configure API URL

npm run dev
```

**Required environment variables:**

```env
NEXT_PUBLIC_API_URL=http://localhost:3000
NEXT_PUBLIC_STELLAR_NETWORK=testnet
```

> **Note:** Next.js only exposes environment variables with the `NEXT_PUBLIC_` prefix to the browser. Use `.env.local` for local overrides.

The frontend runs at `http://localhost:3000` (or the port shown in terminal).

## Stellar Testnet Setup

### Fund Your Test Account

```bash
# Generate a testnet keypair
soroban keys generate testuser --network testnet

# Fund it with testnet XLM via Friendbot
curl "https://friendbot.stellar.org?addr=$(soroban keys address testuser)"
```

### Install Freighter Wallet

1. Install the [Freighter browser extension](https://freighter.app/)
2. Switch to Testnet network in settings
3. Import your test account using the secret key

### Get Test LEARN Tokens

After deploying contracts, mint test tokens:

```bash
soroban contract invoke \
  --id $LEARN_TOKEN_ADDRESS \
  --source deployer \
  --network testnet \
  -- \
  mint \
  --to $(soroban keys address testuser) \
  --amount 10000000000  # 1000 LEARN
```

## Development Workflow

### Branch Naming
- `feature/description` — new features
- `fix/description` — bug fixes
- `test/description` — test additions
- `docs/description` — documentation changes

### Commit Messages
Follow [Conventional Commits](https://www.conventionalcommits.org/):
- `feat: add quiz generation endpoint`
- `fix: resolve enrollment count bug`
- `test: add E2E tests for credentials module`
- `docs: update local development guide`

### PR Process
1. Create a branch from `main`
2. Make changes with tests
3. Run lint + typecheck:
   - API: `npm run lint && npm run typecheck`
   - Frontend: `npm run lint`
   - AI: `ruff check src/ tests/`
   - Contracts: `cargo test`
4. Push and create PR
5. PR will trigger CI (lint, test, build)
6. Address review feedback
7. Squash and merge

### Code Style
- TypeScript: ESLint + Prettier (existing config)
- Python: Ruff (line length 100)
- Rust: `cargo fmt` + `cargo clippy`
- Commit messages: Conventional Commits

### Running Tests

```bash
# API tests
cd chainlearn-api && npm test

# AI service tests
cd chainlearn-ai && pytest

# Contract tests
cd chainlearn-contracts && cargo test

# Frontend — no test script; run lint instead
cd chainlearn-frontend && npm run lint

# Integration tests (requires all services running)
cd chainlearn-infra && docker compose -f docker-compose.test.yml up
```

### Common Tasks

**Reset the database:**
```bash
cd chainlearn-api
npm run db:generate
npm run db:migrate
npm run db:seed
```

**Clear Redis cache:**
```bash
redis-cli FLUSHDB
```

**View contract events:**
```bash
soroban events --id $LEARN_TOKEN_ADDRESS --network testnet
```

## Troubleshooting

### Port Conflicts

If a port is already in use:

```bash
# Find the process
lsof -i :3000

# Kill it
kill -9 <PID>
```

Or change the port in the service's `.env` file.

### Database Connection Errors

Ensure PostgreSQL is running:
```bash
docker compose up -d postgres
docker compose logs postgres
```

### Contract Deployment Failures

- Ensure the deployer account is funded: `soroban keys fund deployer --network testnet`
- Check WASM size: `ls -la target/wasm32-unknown-unknown/release/*.wasm`
- Verify network: `soroban network ls`

### "Module not found" errors in contracts

Ensure the WASM target is installed:
```bash
rustup target add wasm32-unknown-unknown
```

### AI Service Errors

- Verify your Cohere API key is valid and has quota:
  ```bash
  curl -H "Authorization: Bearer $COHERE_API_KEY" https://api.cohere.ai/v1/models
  ```
- Check Redis is running (AI service uses it for caching)
- Look at logs: `docker compose logs chainlearn-ai`

### Frontend can't connect to API

Check that `NEXT_PUBLIC_API_URL` is set in `.env.local` (not `.env`).
Next.js only exposes env vars with `NEXT_PUBLIC_` prefix to the browser.

## IDE Setup

### VS Code

Recommended extensions:
- rust-analyzer (for contracts)
- Python (for AI service)
- ESLint + Prettier (for API and frontend)
- Drizzle ORM (for database schema)
- Soroban (for contract development)

### Workspace Settings

Create `.vscode/settings.json` in your workspace root:

```json
{
  "rust-analyzer.linkedProjects": [
    "chainlearn-contracts/Cargo.toml"
  ],
  "python.defaultInterpreterPath": "chainlearn-ai/venv/bin/python",
  "editor.formatOnSave": true
}
```
