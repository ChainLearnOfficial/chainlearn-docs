# Stellar Integration Guide

This guide covers the Stellar-specific patterns and practices used throughout ChainLearn. It is intended for developers working on the API, indexer, or any service that interacts with the Stellar network.

## Overview

ChainLearn uses Stellar for three purposes:

1. **SEP-10 Authentication** -- Wallet-based login using Stellar keypairs
2. **SEP-41 Tokens** -- LEARN reward token distribution
3. **Soroban Contracts** -- Credential NFTs and progress tracking

## SEP-10: Stellar Authentication

SEP-10 defines a challenge-response authentication flow that proves a user controls a Stellar keypair without exposing their secret key.

### How It Works

```
1. Client requests a challenge    ->  GET /auth/challenge?account=GABC...
2. Server builds a challenge tx   <-  Transaction with random sequence, timebounds
3. Client signs the challenge     ->  User signs with Freighter/Albedo
4. Client submits signed tx       ->  POST /auth/verify
5. Server verifies signature      <-  JWT issued
```

### Implementation (API Side)

```typescript
import {
  Keypair,
  Networks,
  TransactionBuilder,
  Account,
  Operation,
} from '@stellar/stellar-sdk';

// Step 1: Build challenge transaction
async function buildChallenge(clientAccount: string): Promise<string> {
  const serverKeypair = Keypair.fromSecret(process.env.STELLAR_SECRET_KEY!);
  const serverAccount = await server.loadAccount(serverKeypair.publicKey());

  const tx = new TransactionBuilder(serverAccount, {
    fee: '100',
    networkPassphrase: Networks.TESTNET,
    timebounds: {
      minTime: Math.floor(Date.now() / 1000),
      maxTime: Math.floor(Date.now() / 1000) + 300, // 5 min expiry
    },
  })
    .addOperation(
      Operation.manageData({
        name: 'chainlearn_auth',
        value: randomBytes(32),
        source: clientAccount,
      })
    )
    .addOperation(
      Operation.manageData({
        name: 'chainlearn_auth',
        value: randomBytes(32),
      })
    )
    .build();

  return tx.toEnvelope().toXDR('base64');
}

// Step 2: Verify signed challenge
async function verifyChallenge(signedXdr: string): Promise<string | null> {
  const tx = new Transaction(signedXdr, Networks.TESTNET);

  // Verify the transaction is valid
  // Verify timebounds haven't expired
  // Verify the client signature matches the claimed account
  // Issue a JWT if valid

  const clientKeypair = tx.signatures[0]; // Simplified
  return issueJwt(clientKeypair.publicKey());
}
```

### Implementation (Frontend Side)

```typescript
import { signTransaction } from '@stellar/freighter-api';

// Step 1: Get challenge
const { transaction } = await fetch('/auth/challenge?account=' + publicKey).then(r => r.json());

// Step 2: Sign with Freighter
const signedXdr = await signTransaction(transaction, {
  networkPassphrase: 'Test SDF Network ; September 2015',
});

// Step 3: Verify and get JWT
const { token } = await fetch('/auth/verify', {
  method: 'POST',
  body: JSON.stringify({ transaction: signedXdr }),
}).then(r => r.json());
```

### Security Considerations

- Challenge transactions expire after 5 minutes
- Each challenge can only be used once (server tracks used nonces)
- The server verifies signatures against the Stellar network, not a local database
- JWTs have a 24-hour TTL with refresh token rotation

## SEP-41: Fungible Token Standard

SEP-41 is Stellar's native fungible token standard for Soroban contracts. LearnToken implements this standard.

### Key Functions

```typescript
import { Contract, Address, nativeToScVal } from '@stellar/stellar-sdk';

const learnToken = new Contract(process.env.LEARN_TOKEN_ADDRESS!);

// Query balance
const balance = await learnToken.call('balance', address);

// Transfer tokens (requires auth from 'from' address)
// This is typically done server-side with the admin key
const tx = new TransactionBuilder(adminAccount, { fee: BASE_FEE, networkPassphrase })
  .addOperation(
    learnToken.call(
      'transfer',
      nativeToScVal(adminAddress, { type: 'address' }),
      nativeToScVal(learnerAddress, { type: 'address' }),
      nativeToScVal(amount, { type: 'i128' })
    )
  )
  .setTimeout(30)
  .build();

tx.sign(adminKeypair);
await server.sendTransaction(tx);
```

### Token Transfer Flow in ChainLearn

```
1. Learner completes quiz with passing score
2. API determines reward amount (from course config)
3. API builds a Soroban transaction calling LearnToken.transfer()
4. API signs with the platform's admin key
5. API submits to Stellar network
6. API records the tx_hash in PostgreSQL
7. Frontend shows the reward with a link to the transaction
```

### Amount Representation

SEP-41 uses i128 for amounts with 7 decimal places (matching Stellar's native precision):

| Human Amount | On-Chain Amount (i128) |
|-------------|----------------------|
| 1 LEARN | 10,000,000 |
| 100 LEARN | 1,000,000,000 |
| 0.5 LEARN | 5,000,000 |

```typescript
function learnToStroops(learn: number): bigint {
  return BigInt(Math.round(learn * 10_000_000));
}

function stroopsToLearn(stroops: bigint): number {
  return Number(stroops) / 10_000_000;
}
```

## Soroban Events

Soroban contracts emit events that the indexer subscribes to. Understanding the event model is critical for building the indexer.

### Event Structure

Each event has:
- **Contract ID** -- The address of the emitting contract
- **Topics** -- Up to 4 SCV values for filtering (like indexed fields in Ethereum)
- **Data** -- An arbitrary SCV value (the event payload)

### Events Emitted by ChainLearn Contracts

**LearnToken:**
```
Event: transfer
Topics: [from_address, to_address]
Data: amount (i128)
```

**CredentialNFT:**
```
Event: credential_minted
Topics: [holder_address, course_id (string)]
Data: { token_id (u64), metadata_uri (string) }
```

**ProgressTracker:**
```
Event: enrolled
Topics: [user_address, course_id (string)]
Data: { total_modules (u32), timestamp (u64) }

Event: module_completed
Topics: [user_address, course_id (string), module_id (string)]
Data: { modules_done (u32), timestamp (u64) }
```

### Subscribing to Events (Indexer)

```typescript
import { SorobanRpc } from '@stellar/stellar-sdk';

const server = new SorobanRpc.Server('https://soroban-testnet.stellar.org');

// Get events since a specific ledger
const events = await server.getEvents({
  startLedger: lastProcessedLedger,
  filters: [
    {
      contractIds: [LEARN_TOKEN_ADDRESS, CREDENTIAL_NFT_ADDRESS, PROGRESS_TRACKER_ADDRESS],
    },
  ],
  limit: 100,
});

for (const event of events.events) {
  // Parse and index the event
  await processEvent(event);
}
```

### Event Processing Pattern

```typescript
async function processEvent(event: SorobanRpc.Api.Event): Promise<void> {
  const contractId = event.contractId;

  switch (contractId) {
    case LEARN_TOKEN_ADDRESS:
      await processTokenEvent(event);
      break;
    case CREDENTIAL_NFT_ADDRESS:
      await processCredentialEvent(event);
      break;
    case PROGRESS_TRACKER_ADDRESS:
      await processProgressEvent(event);
      break;
  }
}
```

## Stellar SDK Best Practices

### Transaction Building

```typescript
import {
  TransactionBuilder,
  Networks,
  BASE_FEE,
  SorobanRpc,
} from '@stellar/stellar-sdk';

// Always use the latest account state
const account = await server.getAccount(adminPublicKey);

const tx = new TransactionBuilder(account, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(/* ... */)
  .setTimeout(30)  // Always set a timeout
  .build();

// Simulate before submitting (Soroban transactions)
const simResult = await server.simulateTransaction(tx);
if (SorobanRpc.Api.isSimulationError(simResult)) {
  throw new Error(`Simulation failed: ${simResult.error}`);
}

// Sign and submit
tx.sign(adminKeypair);
const result = await server.sendTransaction(tx);

// Wait for confirmation
const hash = result.hash;
let status = result.status;
while (status === 'PENDING' || status === 'NOT_FOUND') {
  await new Promise(resolve => setTimeout(resolve, 1000));
  status = (await server.getTransaction(hash)).status;
}
```

### Error Handling

```typescript
import { SorobanRpc } from '@stellar/stellar-sdk';

try {
  const result = await server.sendTransaction(tx);

  if (result.status === 'ERROR') {
    // Parse the error
    console.error('Transaction failed:', result.errorResult);
    // Handle specific error codes from contracts
  }
} catch (error) {
  if (error instanceof SorobanRpc.Api.SendTransactionError) {
    // Network-level error
  }
}
```

### Account Sequence Management

Stellar requires sequence numbers for transaction ordering. When submitting multiple transactions rapidly:

```typescript
// Load account fresh each time (sequence number increments)
const account = await server.getAccount(adminPublicKey);

// Or manage sequence manually for rapid submissions
let sequence = BigInt(account.sequenceNumber());
for (const operation of batch) {
  account.sequenceNumber = (sequence++).toString();
  const tx = new TransactionBuilder(account, { ... })
    .addOperation(operation)
    .build();
  tx.sign(adminKeypair);
  await server.sendTransaction(tx);
}
```

## Stellar Network Configuration

| Setting | Testnet | Mainnet |
|---------|---------|---------|
| Network Passphrase | `Test SDF Network ; September 2015` | `Public Global Stellar Network ; September 2015` |
| Horizon RPC | `https://horizon-testnet.stellar.org` | `https://horizon.stellar.org` |
| Soroban RPC | `https://soroban-testnet.stellar.org` | `https://soroban.stellar.org` |
| Friendbot | `https://friendbot.stellar.org` | N/A |
| Explorer | `https://stellar.expert/explorer/testnet` | `https://stellar.expert/explorer/public` |

## Common Patterns

### Retry with Exponential Backoff

```typescript
async function submitWithRetry(tx: Transaction, maxRetries = 3): Promise<string> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const result = await server.sendTransaction(tx);
      if (result.status !== 'ERROR') return result.hash;

      // If duplicate sequence, refresh account and rebuild
      if (result.errorResult?.resultCodes()?.transactionResult === 'tx_bad_seq') {
        const account = await server.getAccount(tx.source);
        // Rebuild transaction with new sequence
        continue;
      }
    } catch (error) {
      if (attempt === maxRetries - 1) throw error;
      await new Promise(r => setTimeout(r, 1000 * Math.pow(2, attempt)));
    }
  }
  throw new Error('Max retries exceeded');
}
```

### Fee Bumping

When the network is congested, increase the fee:

```typescript
import { FeeBumpTransaction, BASE_FEE } as sdk from '@stellar/stellar-sdk';

const bumped = new FeeBumpTransaction(feeSourceKeypair, {
  fee: (BigInt(BASE_FEE) * 2n).toString(), // 2x base fee
  innerTransaction: originalTx,
});
bumped.sign(feeSourceKeypair);
await server.sendTransaction(bumped);
```
