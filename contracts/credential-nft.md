# Credential NFT Contract

The CredentialNFT contract manages non-transferable (soulbound) NFTs that represent course completion credentials. Each credential is permanently bound to the learner's Stellar address and cannot be transferred, sold, or traded.

## Contract Address

| Network | Address |
|---------|---------|
| Testnet | `CBXNM...TODO` |
| Mainnet | `TODO` |

## Design Philosophy

Credentials in ChainLearn are **soulbound tokens (SBTs)**. This design choice is intentional:

- A credential represents **personal achievement**, not a tradeable asset
- Transferability would undermine the value of credentials as proof of learning
- Employers and institutions verifying credentials need to trust that the holder earned it
- The credential is permanently tied to the learner's on-chain identity

## Functions

### Admin Functions

#### `mint(to: Address, course_id: String, metadata_uri: String) -> u64`

Mints a new credential NFT to the specified address. Returns the credential's token ID.

```rust
let token_id = credential_nft.mint(
    &learner_address,
    "course_soroban_101",
    "ipfs://Qm...metadata"
);
```

**Access:** Admin only (platform backend)

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `to` | Address | Learner's Stellar address |
| `course_id` | String | Unique course identifier |
| `metadata_uri` | String | URI to off-chain metadata (IPFS or API) |

**Events emitted:**
- `credential_minted(token_id, to, course_id, metadata_uri)`

**Constraints:**
- Each address can hold at most one credential per course (prevents duplicate minting)
- The `to` address must be a valid Stellar account

#### `revoke(token_id: u64, reason: String)`

Revokes a credential. The NFT remains in the holder's address but is marked as revoked. Verification queries will return `revoked: true`.

```rust
credential_nft.revoke(&token_id, "Fraudulent completion detected");
```

**Access:** Admin only (requires multisig approval)

**Events emitted:**
- `credential_revoked(token_id, reason)`

### Public Functions

#### `get_credential(token_id: u64) -> Credential`

Returns the full credential record for a given token ID.

```rust
let credential = credential_nft.get_credential(&token_id);
// Returns: Credential { holder, course_id, metadata_uri, minted_at, revoked, revoked_reason }
```

**Returns:**
```rust
struct Credential {
    holder: Address,
    course_id: String,
    metadata_uri: String,
    minted_at: u64,       // Ledger close time
    revoked: bool,
    revoked_reason: String,
}
```

#### `get_credentials_for(holder: Address) -> Vec<u64>`

Returns all credential token IDs held by an address.

```rust
let token_ids = credential_nft.get_credentials_for(&learner_address);
// Returns: [1, 5, 12] -- list of credential IDs
```

#### `verify(token_id: u64) -> VerificationResult`

Verifies a credential's existence and validity. Returns a structured result suitable for verification interfaces.

```rust
let result = credential_nft.verify(&token_id);
// Returns: VerificationResult { valid, holder, course_id, minted_at }
```

**Returns:**
```rust
struct VerificationResult {
    valid: bool,          // true if exists and not revoked
    holder: Address,
    course_id: String,
    minted_at: u64,
}
```

#### `has_credential(holder: Address, course_id: String) -> bool`

Checks if an address holds a credential for a specific course.

```rust
let has_it = credential_nft.has_credential(&learner_address, "course_soroban_101");
// Returns: true or false
```

#### `total_credentials() -> u64`

Returns the total number of credentials minted.

#### `name() -> String`

Returns the contract name: `"ChainLearn Credential"`

#### `symbol() -> String`

Returns the contract symbol: `"CLCRED"`

## Events

| Event | Topics | Data | Description |
|-------|--------|------|-------------|
| `credential_minted` | `holder`, `course_id` | `token_id`, `metadata_uri` | New credential issued |
| `credential_revoked` | `token_id` | `reason` | Credential revoked |

## Metadata Schema

Credential metadata is stored off-chain (IPFS preferred, API fallback) and follows the schema.org `EducationalOccupationalCredential` schema for interoperability.

### Metadata JSON Structure

```json
{
  "@context": "https://schema.org",
  "@type": "EducationalOccupationalCredential",
  "name": "Stellar Smart Contract Development",
  "description": "Awarded for completing the ChainLearn course on Soroban smart contract development with a passing score.",
  "credentialCategory": "Course Completion",
  "recognizedBy": {
    "@type": "Organization",
    "name": "ChainLearn",
    "url": "https://chainlearn.dev"
  },
  "dateIssued": "2024-06-15T14:30:00Z",
  "expires": null,
  "image": "https://assets.chainlearn.dev/credentials/soroban-101.png",
  "achievement": {
    "course_id": "course_soroban_101",
    "course_title": "Stellar Smart Contract Development",
    "score": 92,
    "modules_completed": 8,
    "total_modules": 8,
    "difficulty": "intermediate"
  },
  "blockchain": {
    "network": "stellar",
    "contract_address": "CBXNM...TODO",
    "token_id": 42,
    "tx_hash": "abc123..."
  }
}
```

### Metadata Hosting

| Priority | Location | Fallback |
|----------|----------|----------|
| Primary | IPFS (pinned via Pinata) | `ipfs://Qm...` |
| Fallback | ChainLearn API | `https://api.chainlearn.dev/credentials/{id}/metadata` |

IPFS is preferred for permanence and decentralization. The API fallback ensures availability if IPFS gateways are slow.

## Verification Flow

Anyone can verify a credential through multiple methods:

### Via ChainLearn UI

1. Visit `https://chainlearn.dev/verify/{credential_id}`
2. The page fetches credential data from the indexer (fast) and cross-references with the contract (authoritative)
3. Displays holder address, course title, score, and issuance date

### Via Stellar Explorer

1. Look up the CredentialNFT contract address on Stellar Expert
2. Call `get_credential(token_id)` or `verify(token_id)`
3. Parse the on-chain response

### Via API

```bash
curl https://api.chainlearn.dev/v1/credentials/{credential_id}
```

Returns the credential metadata with an `on_chain_verified` boolean.

### Programmatic Verification

```typescript
import { Contract, SorobanRpc } from '@stellar/stellar-sdk';

const contract = new Contract(CREDENTIAL_NFT_ADDRESS);
const result = await contract.call('verify', tokenId);

if (result.valid) {
  console.log(`Credential verified for ${result.holder}`);
  console.log(`Course: ${result.course_id}`);
  console.log(`Issued: ${new Date(result.minted_at * 1000)}`);
} else {
  console.log('Credential not found or revoked');
}
```

## Security Considerations

**Non-transferability.** The contract has no `transfer` or `transfer_from` functions. This is enforced at the contract level, not just by convention.

**Duplicate prevention.** The contract maintains a mapping of `(address, course_id) -> token_id` to prevent minting duplicate credentials for the same course.

**Revocation.** Credentials can be revoked by the admin multisig in cases of fraud (e.g., cheating detected). Revoked credentials remain on-chain for transparency but are flagged as invalid in verification queries.

**Metadata integrity.** The metadata URI is set at mint time and stored on-chain. While the content at the URI can change (if hosted on a mutable server), the URI itself is immutable. IPFS URIs provide stronger integrity guarantees.
