# Data Flow

This document traces the complete lifecycle of a learner's journey through ChainLearn, from account creation through credential minting. Each step identifies which service handles the operation and where data is stored.

## End-to-End Flow

```
Enroll -> Study -> Take Quiz -> Get Score -> Earn Reward -> Mint Credential
  |         |          |            |             |              |
 API      API+AI      API+AI       API       API+Contracts   API+Contracts
 PG        PG/Cache    PG          PG          Stellar         Stellar+PG
```

## Step 1: User Registration and Wallet Linking

**Service:** `chainlearn-api`

1. User opens `chainlearn-frontend` and clicks "Sign In."
2. Frontend initiates SEP-10 challenge/response flow:
   - Frontend requests a challenge transaction from `chainlearn-api` (`POST /auth/challenge`).
   - User signs the challenge with their Stellar wallet (Freighter/Albedo).
   - Frontend submits the signed challenge to `chainlearn-api` (`POST /auth/verify`).
3. API validates the signature against the Stellar network, issues a JWT.
4. API creates or updates the user record in PostgreSQL.

**Data stored:**
| Location | Data |
|----------|------|
| PostgreSQL | `users` table: `id`, `stellar_address`, `display_name`, `created_at` |
| Client | JWT in localStorage |

## Step 2: Course Enrollment

**Service:** `chainlearn-api`

1. User browses the course catalog on the frontend.
2. Frontend fetches courses from `GET /courses` (paginated, filterable).
3. User selects a course and clicks "Enroll."
4. Frontend calls `POST /courses/{id}/enroll`.
5. API creates an enrollment record and calls the ProgressTracker contract to register enrollment on-chain.
6. API calls `POST /courses/{id}/modules` to fetch the module list (content may be AI-generated on first access).

**Data stored:**
| Location | Data |
|----------|------|
| PostgreSQL | `enrollments` table: `id`, `user_id`, `course_id`, `status`, `enrolled_at` |
| PostgreSQL | `modules` table: `id`, `course_id`, `title`, `content`, `order` |
| Stellar (ProgressTracker) | Enrollment event emitted with `user`, `course_id`, `timestamp` |

## Step 3: Content Study

**Service:** `chainlearn-api` + `chainlearn-ai`

1. User opens a module. Frontend fetches content from `GET /modules/{id}`.
2. If module content exists in PostgreSQL, API returns it directly.
3. If content needs generation (first access or refresh):
   - API sends a request to `chainlearn-ai` (`POST /generate/course`).
   - AI service generates structured content using Cohere's language model.
   - AI service returns the content to the API.
   - API persists the content in PostgreSQL and returns it to the frontend.
4. User reads the content. Frontend tracks reading progress client-side.
5. When the user completes a module, frontend calls `POST /modules/{id}/complete`.
6. API updates the enrollment progress and calls ProgressTracker to log module completion on-chain.

**Data stored:**
| Location | Data |
|----------|------|
| PostgreSQL | `module_completions` table: `id`, `user_id`, `module_id`, `completed_at` |
| PostgreSQL | `modules.content` | AI-generated content cached here |
| Redis | AI generation cache (TTL: 24h) |
| Stellar (ProgressTracker) | Module completion event |

## Step 4: Quiz Taking

**Service:** `chainlearn-api` + `chainlearn-ai`

1. After completing all modules, user unlocks the course quiz.
2. Frontend calls `POST /quizzes/generate` with the course ID.
3. API forwards the request to `chainlearn-ai` (`POST /generate/quiz`).
4. AI service generates quiz questions based on the course content, calibrated to the course difficulty level.
5. API stores the quiz in PostgreSQL and returns it to the frontend (answers omitted from response).
6. User answers each question. Frontend calls `POST /quizzes/{id}/submit` with all answers.
7. API sends the submission to `chainlearn-ai` (`POST /evaluate/quiz`).
8. AI service evaluates each answer, providing a score and per-question feedback.
9. API stores the result and checks if the score meets the passing threshold (default: 70%).

**Data stored:**
| Location | Data |
|----------|------|
| PostgreSQL | `quizzes` table: `id`, `course_id`, `questions` (JSONB), `created_at` |
| PostgreSQL | `quiz_attempts` table: `id`, `quiz_id`, `user_id`, `score`, `answers` (JSONB), `feedback` (JSONB), `passed`, `submitted_at` |
| Redis | Quiz cache (TTL: 1h) |

## Step 5: Reward Distribution

**Service:** `chainlearn-api` + `chainlearn-contracts`

1. If the quiz score meets the passing threshold, the API initiates reward distribution.
2. API calls the LearnToken contract's `transfer` function to send tokens to the learner's Stellar address.
3. The amount is determined by the course's reward configuration (e.g., 100 LEARN tokens per course).
4. API records the reward transaction in PostgreSQL.
5. Frontend displays the reward with a link to the Stellar transaction.

**Data stored:**
| Location | Data |
|----------|------|
| PostgreSQL | `rewards` table: `id`, `user_id`, `course_id`, `amount`, `tx_hash`, `distributed_at` |
| Stellar (LearnToken) | Token transfer from platform treasury to learner |
| Stellar ledger | Transaction record with memo |

## Step 6: Credential Minting

**Service:** `chainlearn-api` + `chainlearn-contracts`

1. After reward distribution, API initiates credential minting.
2. API calls the CredentialNFT contract's `mint` function with:
   - `to`: learner's Stellar address
   - `course_id`: the completed course
   - `metadata_uri`: pointer to off-chain metadata (IPFS or API-hosted JSON)
3. The NFT is non-transferable (soulbound). The contract enforces this at the code level.
4. API records the credential in PostgreSQL with the mint transaction hash.
5. Frontend shows the credential in the user's credential gallery.

**Data stored:**
| Location | Data |
|----------|------|
| PostgreSQL | `credentials` table: `id`, `user_id`, `course_id`, `nft_id`, `tx_hash`, `metadata_uri`, `minted_at` |
| Stellar (CredentialNFT) | NFT minted to learner's address |
| IPFS / API | Credential metadata JSON: `{ course_title, completion_date, score, issuer }` |

## Step 7: Credential Verification

**Service:** `chainlearn-api` + `chainlearn-indexer`

1. Anyone can verify a credential by querying the Stellar network.
2. A verifier visits `chainlearn-frontend/verify/{credential_id}` or queries the API directly.
3. API (or frontend via indexer) reads the CredentialNFT contract's `get_credential` function.
4. Indexer provides fast lookup of credential metadata by caching on-chain data in PostgreSQL.
5. The verification result shows: holder address, course title, completion date, score, and issuer signature.

**Data read:**
| Location | Data |
|----------|------|
| Stellar (CredentialNFT) | On-chain credential record (authoritative) |
| PostgreSQL (indexer) | Cached credential metadata for fast queries |

## Error Handling

| Failure Point | Behavior |
|---------------|----------|
| AI service unavailable | Course content falls back to pre-authored templates. Quiz generation retries 3x, then queues for later. |
| Smart contract call fails | Reward/credential is queued in a `pending_rewards` table. A background job retries every 5 minutes for 24 hours. |
| Stellar network congestion | Transactions use increasing fee bumps. After 3 retries, the operation is queued. |
| Indexer lag | API reads directly from Stellar RPC for real-time queries. Indexer data is eventually consistent (< 30s lag). |
