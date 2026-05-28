# ADR-001: Use Soroban/Stellar Instead of Ethereum

- **Status:** Accepted
- **Date:** 2024-01-15
- **Deciders:** ChainLearn core team

## Context

ChainLearn needs a blockchain platform for issuing reward tokens and verifiable credentials. The two primary candidates are:

1. **Ethereum (L1 or L2)** -- The dominant smart contract platform with the largest ecosystem, developer tooling, and user base.
2. **Stellar / Soroban** -- A purpose-built payments and asset platform with native smart contract support via Soroban.

The decision affects token economics, user experience, development cost, and long-term platform viability.

### Requirements

- Low transaction fees (micro-rewards of < $1 must be viable)
- Fast finality (learners should see rewards within seconds)
- Smart contract capability (custom token logic, NFTs, state tracking)
- Developer-friendly tooling
- Regulatory consideration (education credentials may require compliance)
- Wallet accessibility (learners may not be crypto-native)

## Decision

We will build ChainLearn's on-chain layer on **Stellar using Soroban smart contracts**.

## Consequences

### Positive

**Low fees.** Stellar transactions cost fractions of a cent (typically 0.00001 XLM, about $0.000001). This makes micro-rewards economically viable. On Ethereum L1, a simple token transfer costs $1-20 depending on congestion, making a 50-LEARN reward impractical. Even on L2s (Optimism, Arbitrum), fees are $0.01-0.10, which adds up over thousands of learners.

**Fast finality.** Stellar confirms transactions in 3-5 seconds. Ethereum L1 takes 12+ seconds for a single confirmation (recommended to wait for 12+ confirmations for finality, taking minutes). L2s are faster but require bridge withdrawals back to L1 that take 7 days.

**Built-in DEX.** Stellar has a native decentralized exchange. If we ever want to list LEARN tokens for trading, the infrastructure exists without deploying a separate DEX contract (like Uniswap on Ethereum).

**SEP standards.** Stellar's SEP (Stellar Ecosystem Proposal) standards provide well-designed primitives:
- SEP-10 for wallet authentication (no need for SIWE or custom auth)
- SEP-41 for fungible tokens (clean, audited standard)
- SEP-20 for contract metadata

**Compliance-friendly.** Stellar was designed with regulatory compliance in mind. The Stellar Development Foundation actively works with regulators. For an education platform that may issue credentials recognized by institutions, this is a significant advantage.

**Soroban is Rust-based.** Rust provides memory safety guarantees that Solidity does not. Smart contract vulnerabilities (reentrancy, overflow) are significantly harder to introduce in Rust.

### Negative

**Smaller ecosystem.** Ethereum has a vastly larger developer community, more DeFi protocols, and more tooling. We will have fewer off-the-shelf libraries and integrations to choose from.

**Less user familiarity.** Most crypto users have MetaMask, not Freighter. The learning curve for Stellar wallets may be steeper for users already in the crypto ecosystem.

**Fewer on-ramps.** Stellar has fewer fiat on-ramps than Ethereum. However, this is less relevant for ChainLearn since learners earn tokens rather than buying them.

**Smaller DeFi ecosystem.** If LEARN tokens need liquidity or DeFi integration, the options are more limited on Stellar. This is acceptable because LEARN is a utility token, not a speculative asset.

### Risks

**Soroban maturity.** Soroban is newer than the Ethereum Virtual Machine. There may be undiscovered edge cases or tooling gaps. Mitigation: We will audit contracts thoroughly and maintain a conservative upgrade path.

**Ecosystem growth.** If Stellar's ecosystem does not grow, ChainLearn's on-chain integrations may become a liability. Mitigation: The on-chain layer is optional -- the platform works with pre-authored content and can migrate to another chain if needed.

## Alternatives Considered

### Ethereum L1

Rejected due to high gas costs. A platform distributing thousands of micro-rewards per day cannot sustain $1-20 per transaction.

### Ethereum L2 (Optimism/Arbitrum)

Strong contender. Lower fees ($0.01-0.10) but still 10-100x more expensive than Stellar. Bridge complexity adds risk. The compliance advantages of Stellar were a differentiator.

### Solana

High throughput and low fees, but has experienced network outages. The Rust-based development model is similar to Soroban, but Stellar's focus on payments and assets aligns better with ChainLearn's use case than Solana's focus on high-frequency DeFi.

### Polygon

Low fees and Ethereum compatibility, but centralization concerns and the complexity of the multi-chain Ethereum ecosystem (which L2 to choose?) made Stellar's simplicity more attractive.
