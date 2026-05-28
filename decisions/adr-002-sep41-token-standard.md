# ADR-002: Use SEP-41 for the Reward Token

- **Status:** Accepted
- **Date:** 2024-01-20
- **Deciders:** ChainLearn core team

## Context

ChainLearn needs a fungible token (LEARN) to reward learners for completing courses. On Stellar, there are two primary options for creating fungible tokens:

1. **Stellar Asset (classic)** -- The native asset type on Stellar's classic layer. Assets are issued by creating a trustline and can be managed with built-in operations.
2. **SEP-41 Token (Soroban)** -- A smart contract-based fungible token standard for Soroban. Similar in concept to ERC-20 on Ethereum but designed for the Soroban execution environment.

The choice affects how tokens are issued, transferred, and integrated with other Soroban contracts.

## Decision

We will implement the LEARN token as a **SEP-41 Soroban smart contract**.

## Consequences

### Positive

**Composability with Soroban contracts.** SEP-41 tokens are first-class Soroban contracts. Our ProgressTracker and CredentialNFT contracts can directly invoke LearnToken transfers in the same transaction. This is critical for the flow where completing a quiz triggers a reward -- all three contracts can be called atomically.

**Programmable token logic.** As a smart contract, we can embed custom logic directly in the token:
- Minting is restricted to the admin (platform treasury)
- Transfers can include custom validation (e.g., pause transfers during emergencies)
- We can add features like time-locked vesting for the team allocation

**Standard interface.** SEP-41 is the standard for Soroban tokens. Every Soroban-compatible wallet, DEX, and tool supports SEP-41. Using a non-standard token would limit integration options.

**Allowance model with expiration.** SEP-41 includes `approve` with mandatory expiration, which is safer than Ethereum's unlimited approvals. This protects users from approval-based attacks.

**Event emission.** SEP-41 emits standard events (`transfer`, `mint`, `burn`) that our indexer can subscribe to without custom parsing.

### Negative

**More complex deployment.** A SEP-41 token requires deploying and initializing a smart contract, whereas a Stellar Asset is created with a simple transaction. The additional complexity is manageable.

**Higher per-transaction cost.** Soroban contract invocations cost more gas than classic Stellar operations. However, Stellar fees are so low that the difference is negligible (still fractions of a cent).

**Contract upgrade risk.** As a smart contract, the token has upgrade potential. While this is useful for bug fixes, it also introduces a trust assumption -- users must trust that the admin won't upgrade the contract maliciously. Mitigation: 48-hour timelock on upgrades, multisig control, and public audit of all upgrades.

### Risks

**Soroban contract bugs.** A bug in the SEP-41 implementation could affect token balances. Mitigation: We use the audited Soroban SDK token implementation rather than writing from scratch.

## Alternatives Considered

### Stellar Asset (Classic)

A Stellar Asset is simpler to create and has lower per-transaction costs. However:

- Cannot be called by other Soroban contracts directly (requires cross-layer bridge)
- No programmable logic (can't add custom minting rules or pause functionality)
- The asset issuer key is a single point of failure (must be kept online or locked)
- Not composable with the Soroban ecosystem

We rejected this option because the composability benefits of SEP-41 outweigh the simplicity of classic assets. The ability to atomically call LearnToken from ProgressTracker is a core architectural requirement.

### Wrapped SEP-41 + Classic Asset

We considered issuing a classic Stellar Asset and wrapping it in a SEP-41 contract for Soroban interactions. This would give us the best of both worlds (classic efficiency, Soroban composability) but adds significant complexity and a potential bridge vulnerability. We decided the direct SEP-41 approach is simpler and safer.

## Implementation Notes

We use the `soroban-sdk`'s built-in token contract implementation (`soroban_token_sdk`) as the base. This provides a tested, audited SEP-41 implementation that we extend with admin controls.

```rust
use soroban_token_sdk::TokenUtils;

#[contractimpl]
impl LearnToken {
    pub fn initialize(env: Env, admin: Address, decimal: u32, name: String, symbol: String) {
        // Uses soroban_token_sdk for standard SEP-41 implementation
        TokenUtils::new(&env).initialize(&admin, &decimal, &name, &symbol);
    }
}
```
