# Progress Tracker Contract

The ProgressTracker contract maintains on-chain records of learner enrollments and module completions. It serves as the authoritative, auditable trail of learning progress on the Stellar network.

## Contract Address

| Network | Address |
|---------|---------|
| Testnet | `CCDNM...TODO` |
| Mainnet | `TODO` |

## Purpose

The ProgressTracker provides three guarantees that off-chain databases cannot:

1. **Auditability** -- Anyone can verify that a learner completed the required modules before receiving a credential.
2. **Composability** -- Other Soroban contracts can read progress data for conditional logic (e.g., scholarship distribution).
3. **Immutability** -- Progress records cannot be retroactively altered, preventing credential fraud.

## Functions

### Admin Functions

#### `enroll(user: Address, course_id: String, total_modules: u32)`

Registers a user's enrollment in a course. Sets up the tracking structure for module completions.

```rust
progress_tracker.enroll(&learner_address, "course_soroban_101", 8);
```

**Access:** Admin only (platform backend)

**Events emitted:**
- `enrolled(user, course_id, total_modules, timestamp)`

**Constraints:**
- Cannot enroll in the same course twice (returns error)
- `total_modules` must be greater than 0

#### `complete_module(user: Address, course_id: String, module_id: String)`

Records a module as completed for a user. Updates the progress counter and checks if all modules are done.

```rust
progress_tracker.complete_module(&learner_address, "course_soroban_101", "module_1_storage");
```

**Access:** Admin only (platform backend)

**Events emitted:**
- `module_completed(user, course_id, module_id, modules_done, timestamp)`

**Constraints:**
- User must be enrolled in the course
- Module must not already be marked as completed
- Module ID must be a valid module for the course

### Public Functions

#### `get_enrollment(user: Address, course_id: String) -> Enrollment`

Returns the enrollment record for a user in a specific course.

```rust
let enrollment = progress_tracker.get_enrollment(&learner_address, "course_soroban_101");
```

**Returns:**
```rust
struct Enrollment {
    user: Address,
    course_id: String,
    total_modules: u32,
    modules_completed: u32,
    enrolled_at: u64,        // Ledger close time
    completed_at: Option<u64>, // None if still in progress
    status: EnrollmentStatus,
}

enum EnrollmentStatus {
    Active,
    Completed,
}
```

#### `get_module_status(user: Address, course_id: String, module_id: String) -> ModuleStatus`

Checks if a specific module is completed.

```rust
let status = progress_tracker.get_module_status(
    &learner_address,
    "course_soroban_101",
    "module_1_storage"
);
```

**Returns:**
```rust
struct ModuleStatus {
    completed: bool,
    completed_at: Option<u64>,
}
```

#### `is_eligible_for_reward(user: Address, course_id: String) -> bool`

Checks if a learner has completed all modules in a course and is eligible for reward distribution.

```rust
let eligible = progress_tracker.is_eligible_for_reward(&learner_address, "course_soroban_101");
// Returns: true if all modules completed
```

This function is called by the API before distributing LEARN tokens and minting credentials.

#### `get_progress_percentage(user: Address, course_id: String) -> u32`

Returns the completion percentage (0-100) for a user's enrollment.

```rust
let progress = progress_tracker.get_progress_percentage(&learner_address, "course_soroban_101");
// Returns: 75 (if 6 of 8 modules completed)
```

#### `get_enrollments_for(user: Address) -> Vec<String>`

Returns all course IDs a user is enrolled in.

```rust
let courses = progress_tracker.get_enrollments_for(&learner_address);
// Returns: ["course_soroban_101", "course_stellar_basics"]
```

#### `get_course_stats(course_id: String) -> CourseStats`

Returns aggregate statistics for a course (public query, no auth required).

```rust
let stats = progress_tracker.get_course_stats("course_soroban_101");
```

**Returns:**
```rust
struct CourseStats {
    course_id: String,
    total_enrollments: u64,
    total_completions: u64,
    completion_rate: u32,  // Percentage (0-100)
}
```

## Events

| Event | Topics | Data | Description |
|-------|--------|------|-------------|
| `enrolled` | `user`, `course_id` | `total_modules`, `timestamp` | User enrolled in course |
| `module_completed` | `user`, `course_id`, `module_id` | `modules_done`, `timestamp` | Module marked complete |
| `course_completed` | `user`, `course_id` | `timestamp` | All modules completed |

## Data Structures

### On-Chain Storage Layout

```
enrollments:
  key: (user_address, course_id) -> Enrollment

module_completions:
  key: (user_address, course_id, module_id) -> u64 (timestamp)

course_stats:
  key: course_id -> CourseStats
```

### Storage Costs

| Operation | Storage | Approximate Cost |
|-----------|---------|-----------------|
| Enroll | ~200 bytes | ~0.00001 XLM |
| Complete module | ~150 bytes | ~0.000008 XLM |
| Read enrollment | Read-only | Free |
| Read module status | Read-only | Free |

Storage costs are negligible on Stellar, making per-module tracking economically viable.

## Integration Flow

The ProgressTracker is called by the ChainLearn API during the learning lifecycle:

```
1. User enrolls
   API -> progress_tracker.enroll(user, course_id, total_modules)

2. User completes a module
   API -> progress_tracker.complete_module(user, course_id, module_id)

3. User finishes quiz
   API -> progress_tracker.is_eligible_for_reward(user, course_id)
   API -> LearnToken.transfer(treasury, user, reward_amount)
   API -> CredentialNFT.mint(user, course_id, metadata_uri)
```

## Error Codes

| Code | Name | Description |
|------|------|-------------|
| 100 | `AlreadyEnrolled` | User is already enrolled in this course |
| 101 | `NotEnrolled` | User is not enrolled in this course |
| 102 | `ModuleAlreadyCompleted` | Module was already marked as completed |
| 103 | `InvalidModule` | Module ID is not valid for this course |
| 104 | `InvalidCourse` | Course has zero total modules |

## Admin Pattern

Only the platform's backend server can call admin functions (`enroll`, `complete_module`). This is a deliberate centralization:

- The API validates that a learner actually completed a module before recording it on-chain
- The on-chain record is the **receipt**, not the verification
- This avoids the complexity of decentralized proof-of-learning while maintaining an auditable trail

### Future: Decentralized Verification

We are exploring adding a peer-review or proof-of-learning layer that would allow the community to verify completions without relying solely on the centralized API. This is tracked in [ADR-001](../decisions/adr-001-soroban-over-ethereum.md) as a future consideration.

## Querying Progress

### From the API

The API maintains a PostgreSQL cache of on-chain progress data (populated by the indexer). Most queries hit PostgreSQL for speed. The on-chain contract is the authoritative source for verification.

### Direct On-Chain Query

```typescript
import { Contract, SorobanRpc } from '@stellar/stellar-sdk';

const provider = new SorobanRpc.Server('https://soroban-testnet.stellar.org');
const contract = new Contract(PROGRESS_TRACKER_ADDRESS);

// Check eligibility
const eligible = await contract.call(
  'is_eligible_for_reward',
  learnerAddress,
  'course_soroban_101'
);

// Get full enrollment details
const enrollment = await contract.call(
  'get_enrollment',
  learnerAddress,
  'course_soroban_101'
);
```

### Via the Indexer

The indexer provides fast, paginated queries over progress data:

```bash
# Get all enrollments for a user
curl https://indexer.chainlearn.dev/v1/progress/enrollments?user=GABC...

# Get detailed progress for an enrollment
curl https://indexer.chainlearn.dev/v1/progress/enrollments/{id}
```
