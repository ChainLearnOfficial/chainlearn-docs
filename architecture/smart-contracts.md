# Smart Contract Design

This document explains the design decisions behind ChainLearn's three Soroban smart contracts: why we have three separate contracts, why we chose SEP-41 for the reward token, and how the credential model works.

## Why Three Contracts

We considered a single monolithic contract but decided on three separate contracts for clear separation of concerns, independent upgradeability, and reduced attack surface per contract.

### Contract Responsibilities

| Contract | Purpose | State |
|----------|---------|-------|
| LearnToken | SEP-41 fungible token for rewards | Token balances, allowances |
| CredentialNFT | Non-transferable NFT for credentials | Credential records per address |
| ProgressTracker | Enrollment and module completion tracking | Enrollment and progress state |

### Rationale

**Separation of concerns.** Each contract has a single responsibility. LearnToken handles tokenomics. CredentialNFT handles identity and verification. ProgressTracker handles learning state. Mixing these would create a contract with complex, intertwined logic that is harder to audit and reason about.

**Independent lifecycle.** If we need to fix a bug in the credential minting logic, we can redeploy CredentialNFT without touching the token contract. Token contracts, once deployed with meaningful supply, should be as immutable as possible.

**Reduced attack surface.** A smaller contract with fewer entry points has a smaller attack surface. The LearnToken contract holds real value (token balances). Keeping enrollment logic out of it reduces the risk of a bug in enrollment code draining token balances.

**Gas efficiency.** Smaller contracts consume less gas per invocation. The ProgressTracker is called frequently (every module completion). Keeping it lean reduces costs for learners.

## SEP-41 Token Standard

### What is SEP-41

SEP-41 is Stellar's native fungible token standard for Soroban smart contracts. It is analogous to ERC-20 on Ethereum but designed specifically for the Soroban execution environment. SEP-41 defines a minimal interface for:

- `transfer(from, to, amount)` -- Move tokens between accounts
- `approve(from, spender, amount, expiration)` -- Set spending allowances
- `balance(address)` -- Query token balance
- `allowance(from, spender)` -- Query spending allowance
- `transfer_from(spender, from, to, amount)` -- Transfer on behalf

### Why SEP-41

**Native standard.** SEP-41 is the canonical token standard on Soroban. Using it means LearnToken is compatible with every Stellar wallet, DEX, and tool that supports SEP-41. This includes Freighter, StellarX, StellarTerm, and any future Soroban DeFi protocol.

**Soroban-native.** SEP-41 tokens are first-class Soroban contracts. They can be called by other Soroban contracts natively, enabling composability. Our ProgressTracker contract can directly invoke LearnToken transfers without cross-chain bridges or oracles.

**Built-in authorization.** SEP-41 includes a robust authorization model with expiring allowances. This is safer than Ethereum's unlimited approvals and protects learners from spending exploits.

**Low fees.** Token transfers on Stellar cost fractions of a cent. This makes micro-rewards (e.g., 0.5 LEARN for completing a single module) economically viable, which is not possible on Ethereum L1.

### Token Configuration

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Name | ChainLearn Token | Clear branding |
| Symbol | LEARN | Short, memorable |
| Decimals | 7 | Stellar native precision (stroops) |
| Initial Supply | 1,000,000,000 LEARN | 1 billion tokens for long-term reward pool |
| Admin | Platform multisig (3-of-5) | Prevents single-point key compromise |

### Tokenomics

- 60% allocated to learner reward pool (distributed over 10+ years)
- 15% for platform development
- 10% for community grants and partnerships
- 10% for initial liquidity (Stellar DEX)
- 5% for team and advisors (4-year vesting)

## Credential NFT Model

### Design Philosophy

Credentials in ChainLearn are **soulbound** -- they are minted to a learner's address and cannot be transferred. This is a deliberate design choice. A credential represents a personal achievement, not a tradeable asset. Making credentials transferable would undermine their value as proof of learning.

### Credential Lifecycle

```
Course Completed -> API Calls mint() -> NFT Created -> Verifiable Forever
                                                    |
                                              Cannot Transfer
```

### Contract Functions

| Function | Access | Description |
|----------|--------|-------------|
| `mint(to, course_id, metadata_uri)` | Admin only | Mint a new credential to a learner |
| `get_credential(token_id)` | Public | Read credential details by ID |
| `get_credentials_for(address)` | Public | List all credentials for an address |
| `verify(token_id)` | Public | Check if a credential exists and is valid |
| `revoke(token_id)` | Admin only | Revoke a credential (e.g., fraud detected) |

### Metadata Structure

Credential metadata is stored off-chain (on IPFS or a hosted API) and referenced by URI. The metadata follows a JSON-LD schema for interoperability:

```json
{
  "@context": "https://schema.org",
  "@type": "EducationalOccupationalCredential",
  "name": "Stellar Smart Contract Development",
  "description": "Completed the ChainLearn course on Soroban smart contract development",
  "credentialCategory": "Course Completion",
  "recognizedBy": {
    "@type": "Organization",
    "name": "ChainLearn"
  },
  "dateIssued": "2024-06-15T14:30:00Z",
  "expires": null,
  "achievement": {
    "course_id": "course_soroban_101",
    "score": 92,
    "modules_completed": 8,
    "total_modules": 8
  }
}
```

### Verification Model

Verification is permissionless. Anyone can query the CredentialNFT contract on Stellar to verify a credential. The verification process:

1. Query `get_credential(token_id)` on the contract.
2. Confirm the credential exists and is not revoked.
3. Optionally fetch and validate the metadata URI.
4. Cross-reference with the ProgressTracker contract to confirm the enrollment and completion events.

This creates a chain of trust: the contract is the source of truth, and the metadata provides human-readable context.

## Progress Tracker Contract

### Purpose

The ProgressTracker maintains on-chain records of learner enrollments and module completions. It serves as the authoritative, auditable trail of learning progress.

### Contract Functions

| Function | Access | Description |
|----------|--------|-------------|
| `enroll(user, course_id)` | Admin only | Register a user's enrollment in a course |
| `complete_module(user, course_id, module_id)` | Admin only | Mark a module as completed |
| `get_enrollment(user, course_id)` | Public | Query enrollment status and progress |
| `get_module_status(user, course_id, module_id)` | Public | Check if a specific module is completed |
| `is_eligible_for_reward(user, course_id)` | Public | Check if all modules are completed |

### Why On-Chain Progress

Progress tracking on-chain provides:

1. **Auditability** -- Anyone can verify that a learner actually completed the required modules before receiving a credential.
2. **Composability** -- Other Soroban contracts can read progress data. A future scholarship contract could automatically award grants to learners who complete specific courses.
3. **Immutability** -- Progress records cannot be retroactively altered, preventing credential fraud.

### Admin Pattern

Only the platform's backend server (via a controlled Stellar account) can call admin functions (`enroll`, `complete_module`). This is a deliberate centralization -- we trust the API to validate that a learner actually completed a module before recording it on-chain. The on-chain record is the receipt, not the verification.

In the future, we may add a decentralized verification layer (e.g., peer review or proof-of-learning) to reduce reliance on the centralized API.

## Contract Interaction Diagram

```
                    +-----------------+
                    |  chainlearn-api |
                    |  (admin signer) |
                    +--------+--------+
                             |
              +--------------+--------------+
              |              |              |
     enroll() |   mint()     |   transfer() |
              |              |              |
     +--------v----+  +-----v------+  +----v----------+
     | Progress    |  | Credential |  | LearnToken    |
     | Tracker     |  | NFT        |  | (SEP-41)      |
     +-------------+  +------------+  +---------------+
              |              |              |
              +--------------+--------------+
                             |
                    Events emitted to
                    Stellar network
                             |
                    +--------v--------+
                    | chainlearn-     |
                    | indexer         |
                    +-----------------+
```

## Security Considerations

**Admin key management.** The admin account that calls `enroll`, `complete_module`, and `mint` is a 3-of-5 multisig. Keys are distributed among senior engineers and stored in hardware wallets.

**Reentrancy.** Soroban's execution model prevents reentrancy by default (no external calls during contract execution). However, we still follow the checks-effects-interactions pattern as defense in depth.

**Overflow protection.** Soroban uses i128 for token amounts, which eliminates overflow for practical token supplies.

**Upgradability.** Contracts are deployed with an upgrade mechanism controlled by the multisig. Upgrades require a 48-hour timelock, giving users time to exit if they disagree with a change.
