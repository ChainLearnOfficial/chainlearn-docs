# Contract Deployment Guide

This guide covers deploying ChainLearn's Soroban smart contracts to Stellar testnet and mainnet.

## Prerequisites

- [Soroban CLI](https://soroban.stellar.org/docs/getting-started/setup) installed (`soroban --version`)
- Rust toolchain with `wasm32-unknown-unknown` target
- Stellar account with funded XLM for deployment
- Access to the multisig key holders for admin initialization

## Build

```bash
cd chainlearn-contracts

# Build all contracts
make build

# Or build individually
cargo build --target wasm32-unknown-unknown --release -p learn-token
cargo build --target wasm32-unknown-unknown --release -p credential-nft
cargo build --target wasm32-unknown-unknown --release -p progress-tracker
```

WASM artifacts are produced at:
```
target/wasm32-unknown-unknown/release/learn_token.wasm
target/wasm32-unknown-unknown/release/credential_nft.wasm
target/wasm32-unknown-unknown/release/progress_tracker.wasm
```

## Test

```bash
# Run all tests
cargo test

# Run with Soroban environment logging
RUST_LOG=soroban_sdk=debug cargo test

# Run integration tests against local sandbox
cargo test --features integration
```

## Testnet Deployment

### Step 1: Configure Testnet Identity

```bash
# Generate or import a deployer key
soroban keys generate deployer --network testnet

# Fund the account with testnet XLM
curl "https://friendbot.stellar.org?addr=$(soroban keys address deployer)"
```

### Step 2: Deploy LearnToken

```bash
# Upload and deploy the contract
soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/learn_token.wasm \
  --source deployer \
  --network testnet

# Note the contract address output (e.g., CAZNM...)

# Initialize the token
soroban contract invoke \
  --id CAZNM... \
  --source deployer \
  --network testnet \
  -- \
  initialize \
  --admin $(soroban keys address deployer) \
  --decimal 7 \
  --name "ChainLearn Token" \
  --symbol "LEARN"
```

### Step 3: Deploy CredentialNFT

```bash
soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/credential_nft.wasm \
  --source deployer \
  --network testnet

soroban contract invoke \
  --id CBXNM... \
  --source deployer \
  --network testnet \
  -- \
  initialize \
  --admin $(soroban keys address deployer) \
  --name "ChainLearn Credential" \
  --symbol "CLCRED"
```

### Step 4: Deploy ProgressTracker

```bash
soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/progress_tracker.wasm \
  --source deployer \
  --network testnet

soroban contract invoke \
  --id CCDNM... \
  --source deployer \
  --network testnet \
  -- \
  initialize \
  --admin $(soroban keys address deployer)
```

### Step 5: Verify Deployment

```bash
# Check LearnToken
soroban contract invoke \
  --id CAZNM... \
  --source deployer \
  --network testnet \
  -- \
  name
# Should return: "ChainLearn Token"

# Check CredentialNFT
soroban contract invoke \
  --id CBXNM... \
  --source deployer \
  --network testnet \
  -- \
  name
# Should return: "ChainLearn Credential"
```

### Step 6: Transfer Admin to Multisig

After deployment, transfer admin control from the deployer key to the platform multisig:

```bash
# Transfer LearnToken admin
soroban contract invoke \
  --id CAZNM... \
  --source deployer \
  --network testnet \
  -- \
  set_admin \
  --new_admin $(soroban keys address multisig)

# Repeat for CredentialNFT and ProgressTracker
```

## Mainnet Deployment

Mainnet deployment follows the same steps as testnet with these differences:

### Network Configuration

```bash
# Use mainnet network
soroban contract deploy --network mainnet ...

# Fund account with real XLM (from exchange or treasury)
```

### Pre-Deployment Checklist

- [ ] All contracts audited by approved auditor
- [ ] Testnet deployment tested for at least 2 weeks
- [ ] Multisig account created and funded (3-of-5 signers verified)
- [ ] Emergency pause mechanism tested
- [ ] Admin key ceremony completed (all 5 key holders present)
- [ ] Deployment runbook reviewed by 2+ engineers
- [ ] Rollback plan documented

### Mainnet Deployment Script

```bash
#!/bin/bash
set -euo pipefail

echo "=== ChainLearn Mainnet Deployment ==="
echo "Deployer: $(soroban keys address deployer)"
echo ""
echo "WARNING: This deploys to MAINNET with real XLM."
read -p "Continue? (yes/no): " confirm
[ "$confirm" = "yes" ] || exit 1

echo "Deploying LearnToken..."
LEARN_TOKEN=$(soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/learn_token.wasm \
  --source deployer \
  --network mainnet)
echo "LearnToken: $LEARN_TOKEN"

echo "Deploying CredentialNFT..."
CRED_NFT=$(soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/credential_nft.wasm \
  --source deployer \
  --network mainnet)
echo "CredentialNFT: $CRED_NFT"

echo "Deploying ProgressTracker..."
PROGRESS=$(soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/progress_tracker.wasm \
  --source deployer \
  --network mainnet)
echo "ProgressTracker: $PROGRESS"

echo ""
echo "=== Deployment Complete ==="
echo "Update .env files with these addresses:"
echo "LEARN_TOKEN_ADDRESS=$LEARN_TOKEN"
echo "CREDENTIAL_NFT_ADDRESS=$CRED_NFT"
echo "PROGRESS_TRACKER_ADDRESS=$PROGRESS"
```

## Contract Upgrades

Contracts are deployed with an upgrade mechanism controlled by the multisig.

### Upgrade Process

1. Build the new WASM binary
2. Run all tests against the new binary
3. Deploy to testnet and verify behavior
4. Submit upgrade transaction (requires 3-of-5 multisig signatures)
5. 48-hour timelock activates (gives users time to exit if they disagree)
6. After timelock, the upgrade is applied

### Upgrade Command

```bash
soroban contract invoke \
  --id <CONTRACT_ADDRESS> \
  --source multisig \
  --network mainnet \
  -- \
  upgrade \
  --wasm_hash <NEW_WASM_HASH>
```

## Environment Variables

After deployment, update the environment configuration:

```env
# .env (chainlearn-api)
STELLAR_NETWORK=testnet
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
LEARN_TOKEN_ADDRESS=CAZNM...
CREDENTIAL_NFT_ADDRESS=CBXNM...
PROGRESS_TRACKER_ADDRESS=CCDNM...
ADMIN_SECRET_KEY=SDNM...  # Platform backend signing key
```

## Troubleshooting

### Common Issues

**"Contract not found" error**
- Verify the contract address is correct
- Check that you're querying the right network (testnet vs mainnet)

**"Insufficient balance" error**
- Fund the deployer account with more XLM
- Check base reserve requirements

**"Auth error" when calling admin functions**
- Ensure the signing key matches the contract admin
- If using multisig, ensure enough signatures are collected

**WASM upload fails**
- Check WASM file size (Stellar has a 64KB limit for contract code)
- Verify the WASM was compiled with `--release`

### Useful Commands

```bash
# Check account balance
soroban keys fund deployer --network testnet

# View contract metadata
soroban contract inspect --id <CONTRACT_ADDRESS> --network testnet

# View contract events
soroban events --id <CONTRACT_ADDRESS> --network testnet
```
