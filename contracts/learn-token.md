# LearnToken Contract

LearnToken (LEARN) is the SEP-41 fungible token used to reward learners on the ChainLearn platform. It is deployed as a Soroban smart contract on the Stellar network.

## Contract Address

| Network | Address |
|---------|---------|
| Testnet | `CAZNM...TODO` |
| Mainnet | `TODO` |

## Token Properties

| Property | Value |
|----------|-------|
| Name | ChainLearn Token |
| Symbol | LEARN |
| Decimals | 7 |
| Total Supply | 1,000,000,000 LEARN |
| Standard | SEP-41 |

## Functions

### Public Functions

#### `balance(address: Address) -> i128`

Returns the LEARN token balance for the given address.

```rust
let balance = token.balance(&address);
// Returns: 1000000000 (100 LEARN with 7 decimals)
```

#### `transfer(from: Address, to: Address, amount: i128)`

Transfers LEARN tokens from one address to another. Requires authorization from the `from` address.

```rust
token.transfer(&from, &to, &amount);
```

**Events emitted:**
- `transfer(from, to, amount)`

#### `approve(from: Address, spender: Address, amount: i128, expiration_ledger: u32)`

Approves a spender to spend tokens on behalf of the `from` address. The approval expires after the specified ledger number.

```rust
token.approve(&from, &spender, &1000000000, &current_ledger + 1000);
```

**Events emitted:**
- `approve(from, spender, amount, expiration_ledger)`

#### `allowance(from: Address, spender: Address) -> i128`

Returns the current allowance for a spender.

```rust
let allowed = token.allowance(&from, &spender);
```

#### `transfer_from(spender: Address, from: Address, to: Address, amount: i128)`

Transfers tokens on behalf of another address. Requires the spender to have sufficient allowance.

```rust
token.transfer_from(&spender, &from, &to, &amount);
```

**Events emitted:**
- `transfer(from, to, amount)`

### Admin Functions

#### `mint(to: Address, amount: i128)`

Mints new LEARN tokens to the specified address. Only callable by the contract admin (platform treasury multisig).

```rust
token.mint(&treasury_address, &10000000000000);
```

**Access:** Admin only (3-of-5 multisig)

**Events emitted:**
- `mint(to, amount)`

#### `burn(from: Address, amount: i128)`

Burns LEARN tokens from the specified address. Requires authorization from the `from` address.

```rust
token.burn(&from, &1000000000);
```

**Events emitted:**
- `burn(from, amount)`

#### `set_admin(new_admin: Address)`

Transfers admin control to a new address. Only callable by the current admin.

**Access:** Admin only

**Events emitted:**
- `set_admin(new_admin)`

## Events

| Event | Topics | Data | Description |
|-------|--------|------|-------------|
| `transfer` | `from`, `to` | `amount` | Token transfer occurred |
| `mint` | `to` | `amount` | New tokens minted |
| `burn` | `from` | `amount` | Tokens burned |
| `approve` | `from`, `spender` | `amount`, `expiration` | Allowance set |

## Tokenomics

### Distribution

| Allocation | Percentage | Amount | Purpose |
|------------|-----------|--------|---------|
| Reward Pool | 60% | 600,000,000 LEARN | Distributed to learners over 10+ years |
| Development | 15% | 150,000,000 LEARN | Platform development fund |
| Community | 10% | 100,000,000 LEARN | Grants, partnerships, ambassadors |
| Liquidity | 10% | 100,000,000 LEARN | Stellar DEX liquidity pools |
| Team | 5% | 50,000,000 LEARN | Team allocation (4-year vesting) |

### Reward Schedule

Rewards are distributed to learners upon successful course completion. The reward amount per course is configured in the platform's database and enforced by the API before calling the contract.

| Course Difficulty | Base Reward | Bonus (score > 90%) |
|-------------------|-------------|---------------------|
| Beginner | 50 LEARN | +25 LEARN |
| Intermediate | 100 LEARN | +50 LEARN |
| Advanced | 200 LEARN | +100 LEARN |

### Vesting Schedule (Team Allocation)

| Period | Unlocked |
|--------|----------|
| Month 0 | 0% |
| Month 12 | 25% (cliff) |
| Month 24 | 50% |
| Month 36 | 75% |
| Month 48 | 100% |

## Integration Guide

### Querying Balance from JavaScript

```typescript
import { Contract, SorobanRpc } from '@stellar/stellar-sdk';

const provider = new SorobanRpc.Server('https://soroban-testnet.stellar.org');
const contract = new Contract(LEARN_TOKEN_ADDRESS);

// Build and simulate the transaction
const tx = contract.call('balance', /* address */);
const result = await provider.simulateTransaction(tx);
```

### Transferring Tokens

```typescript
const tx = new TransactionBuilder(account, {
  fee: BASE_FEE,
  networkPassphrase: Networks.TESTNET,
})
  .addOperation(
    contract.call('transfer', from, to, amount)
  )
  .setTimeout(30)
  .build();

// Sign and submit
tx.sign(keypair);
await provider.sendTransaction(tx);
```

## Security Notes

- The admin key is held by a 3-of-5 multisig to prevent single-point compromise.
- Token transfers require explicit authorization from the sender (SEP-41 standard).
- Allowances have mandatory expiration to prevent unlimited approvals.
- The contract has been audited by [TODO: auditor name] on [TODO: date].
