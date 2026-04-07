# Morpho Blue Full Lending Stack — Environment Variables Guide

> For DevOps deploying the complete Morpho lending stack to Base chain.
> Each variable is explained: what it is, where it comes from, and the exact command to obtain it.

---

## Prerequisites

Before starting, you need three things set up independently:

| What | How to Get |
|------|------------|
| **Foundry** | `curl -L https://foundry.paradigm.xyz \| bash && foundryup` |
| **Funded wallet on Base** | ~0.1 ETH on Base (bridge via bridge.base.org) |
| **BaseScan API key** | Register at basescan.org → My Account → API Keys |

---

## Master .env File

Create `~/morpho/.env` and populate it as you work through each step below. Every deployment step reads from this file via `source ~/morpho/.env`.

```bash
# ═══════════════════════════════════════════════════════════════
# TIER 1 — You provide these BEFORE any deployment
# ═══════════════════════════════════════════════════════════════
BASE_RPC_URL=
DEPLOYER_PRIVATE_KEY=
DEPLOYER_ADDRESS=
BASESCAN_API_KEY=
FEE_RECIPIENT=

# ═══════════════════════════════════════════════════════════════
# TIER 2 — Filled sequentially as each contract deploys
# ═══════════════════════════════════════════════════════════════
MORPHO_ADDRESS=
IRM_ADDRESS=
ORACLE_FACTORY_ADDRESS=
ORACLE_WETH_USDC_ADDRESS=
ORACLE_CBETH_USDC_ADDRESS=
BUNDLER3_ADDRESS=
GENERAL_ADAPTER_ADDRESS=
PUBLIC_ALLOCATOR_ADDRESS=
PRE_LIQUIDATION_FACTORY_ADDRESS=
```

---

## Tier 1: Variables You Provide (Pre-Deployment)

These are not produced by any deployment — you bring them yourself.

### `BASE_RPC_URL`

**What**: JSON-RPC endpoint for Base chain.

**How to get**:
- **Free (rate-limited)**: `https://mainnet.base.org`
- **Paid (recommended)**: Sign up at [alchemy.com](https://alchemy.com) or [infura.io](https://infura.io), create a Base app, copy the URL

**Verify**:
```bash
cast chain-id --rpc-url "$BASE_RPC_URL"
# Expected: 8453 (Base mainnet) or 84532 (Base Sepolia)
```

---

### `DEPLOYER_PRIVATE_KEY`

**What**: Hex-encoded private key of the wallet that will sign all deploy transactions.

**How to get**:
- Generate a new wallet: `cast wallet new`
- Or export from MetaMask: Account Details → Export Private Key

**Format**: Must start with `0x`, 66 characters total.

```bash
# Example (DO NOT use this):
DEPLOYER_PRIVATE_KEY=0x7065e502d20347f3c34d9ce4f658bdf5db5b8c8173d4675610932e1666b7c42a
```

> **SECURITY**: Never commit this to git. Never share it. Use a dedicated deployer wallet, not your main wallet.

---

### `DEPLOYER_ADDRESS`

**What**: The public address corresponding to `DEPLOYER_PRIVATE_KEY`.

**How to get**:
```bash
cast wallet address "$DEPLOYER_PRIVATE_KEY"
# Output: 0x2829FaBFf61D3bb5B52249504a8EdFbd43c5f8cF
```

---

### `BASESCAN_API_KEY`

**What**: API key for verifying contracts on BaseScan (so source code is readable on the explorer).

**How to get**:
1. Go to [basescan.org](https://basescan.org)
2. Sign up / Log in
3. My Account → API Keys → Add
4. Copy the key

**Not strictly required** — deployments work without it, but contracts won't show verified source on the explorer.

---

### `FEE_RECIPIENT`

**What**: Address that receives protocol fees from Morpho markets. This is your choice — not produced by a deployment.

**Common choices**:
- Your deployer address (simplest)
- A multisig / treasury address (production)
- `address(0)` — no fees collected

**Set on-chain after Morpho deploys** (Step 1 below):
```bash
cast send "$MORPHO_ADDRESS" "setFeeRecipient(address)" "$FEE_RECIPIENT" \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY"
```

---

## Tier 2: Variables Produced by Deployment (Sequential)

Each step deploys a contract. The output address becomes an env var used by later steps. **You must follow this order.**

```
Step 1 ─── Morpho.sol ──────────────────────→ MORPHO_ADDRESS
              │
Step 2 ───── AdaptiveCurveIrm(morpho) ─────→ IRM_ADDRESS
              │
Step 3 ───── OracleFactory() ───────────────→ ORACLE_FACTORY_ADDRESS (optional)
              │    └── createOracle(...) ───→ ORACLE_WETH_USDC_ADDRESS
              │    └── createOracle(...) ───→ ORACLE_CBETH_USDC_ADDRESS
              │
Step 4 ───── Bundler3() ───────────────────→ BUNDLER3_ADDRESS
              │
Step 5 ───── GeneralAdapter1(bundler,morpho,weth) → GENERAL_ADAPTER_ADDRESS
              │
Step 6 ───── PublicAllocator(morpho) ──────→ PUBLIC_ALLOCATOR_ADDRESS
              │
Step 7 ───── PreLiquidationFactory(morpho) → PRE_LIQUIDATION_FACTORY_ADDRESS
```

---

### Step 1: `MORPHO_ADDRESS`

**What**: The Morpho Blue singleton — the core lending protocol. Everything else plugs into this.

**Repo**: `morpho-blue`
**Contract**: `src/Morpho.sol`
**Constructor args**: `address newOwner` (use your deployer address)

**Deploy**:
```bash
cd ~/morpho/morpho-blue
source ~/morpho/.env

forge script script/DeployMorpho.s.sol:DeployMorpho \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast --verify \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  -vvv
```

**Get the address**: Read the console output line `Morpho Blue: 0x...`

**Verify**:
```bash
# Should return your DEPLOYER_ADDRESS
cast call "$MORPHO_ADDRESS" "owner()(address)" --rpc-url "$BASE_RPC_URL"
```

**Update .env**: `MORPHO_ADDRESS=0x_PASTE_HERE`

---

### Step 2: `IRM_ADDRESS`

**What**: The Adaptive Curve Interest Rate Model. Calculates borrow/supply rates for every market based on utilization.

**Repo**: `morpho-blue-irm`
**Contract**: `src/adaptive-curve-irm/AdaptiveCurveIrm.sol`
**Constructor args**: `address morpho` → use `MORPHO_ADDRESS` from Step 1
**Depends on**: `MORPHO_ADDRESS`

**Deploy**:
```bash
cd ~/morpho/morpho-blue-irm
source ~/morpho/.env

forge script script/DeployIrm.s.sol:DeployIrm \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast --verify \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  -vvv
```

**Get the address**: Read the console output line `AdaptiveCurveIrm: 0x...`

**Post-deploy — enable the IRM on Morpho** (required, or market creation will revert):
```bash
cast send "$MORPHO_ADDRESS" "enableIrm(address)" "$IRM_ADDRESS" \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY"
```

**Verify**:
```bash
# Should return true
cast call "$MORPHO_ADDRESS" "isIrmEnabled(address)(bool)" "$IRM_ADDRESS" \
  --rpc-url "$BASE_RPC_URL"
```

**Update .env**: `IRM_ADDRESS=0x_PASTE_HERE`

---

### Step 3: `ORACLE_FACTORY_ADDRESS` (optional)

**What**: Factory that deploys Chainlink-based oracle adapters via CREATE2.

**Repo**: `morpho-blue-oracles`
**Contract**: `src/morpho-chainlink/MorphoChainlinkOracleV2Factory.sol`
**Constructor args**: None
**Depends on**: Nothing

> **You can skip this step** if you deploy oracle contracts directly or use a mock oracle (see `morpho-blue/src/mocks/OracleMock.sol`).

**Deploy**:
```bash
cd ~/morpho/morpho-blue-oracles
source ~/morpho/.env

forge create src/morpho-chainlink/MorphoChainlinkOracleV2Factory.sol:MorphoChainlinkOracleV2Factory \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --verify --etherscan-api-key "$BASESCAN_API_KEY"
```

**Update .env**: `ORACLE_FACTORY_ADDRESS=0x_PASTE_HERE`

#### Creating oracle instances (e.g., WETH/USDC)

After the factory is deployed, call it to create oracles for each token pair. Each `createMorphoChainlinkOracleV2` call returns a new oracle address.

**Base mainnet Chainlink feeds you'll reference**:

| Feed | Address | Decimals |
|------|---------|----------|
| ETH/USD | `0x71041dddad3595F9CEd3DcCFBe3D1F4b0a16Bb70` | 8 |
| USDC/USD | `0x7e860098F58bBFC8648a4311b374B1D669a2bc6B` | 8 |
| cbETH/ETH | `0x806b4Ac04501c29769051e42783cF04dCE41440b` | 18 |

> Verify feed addresses at [docs.chain.link/data-feeds/price-feeds/addresses](https://docs.chain.link/data-feeds/price-feeds/addresses) before deploying.

**Update .env**: `ORACLE_WETH_USDC_ADDRESS=0x...`, `ORACLE_CBETH_USDC_ADDRESS=0x...`

---

### Step 4: `BUNDLER3_ADDRESS`

**What**: Transaction bundler that batches multiple Morpho operations into a single transaction. Uses EIP-1153 transient storage for reentrancy safety.

**Repo**: `bundler3`
**Contract**: `src/Bundler3.sol`
**Constructor args**: None (stateless, uses transient storage)
**Depends on**: Nothing

**Deploy**:
```bash
cd ~/morpho/bundler3
source ~/morpho/.env

forge create src/Bundler3.sol:Bundler3 \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --verify --etherscan-api-key "$BASESCAN_API_KEY"
```

**Update .env**: `BUNDLER3_ADDRESS=0x_PASTE_HERE`

---

### Step 5: `GENERAL_ADAPTER_ADDRESS`

**What**: Adapter that lets Bundler3 interact with Morpho (supply, borrow, repay, etc.) and wrap/unwrap native ETH.

**Repo**: `bundler3`
**Contract**: `src/adapters/GeneralAdapter1.sol`
**Constructor args**:
  - `address bundler3` → `BUNDLER3_ADDRESS` from Step 4
  - `address morpho` → `MORPHO_ADDRESS` from Step 1
  - `address wNative` → WETH on Base: `0x4200000000000000000000000000000000000006`
**Depends on**: `BUNDLER3_ADDRESS`, `MORPHO_ADDRESS`

**Deploy**:
```bash
cd ~/morpho/bundler3
source ~/morpho/.env

WNATIVE=0x4200000000000000000000000000000000000006

forge create src/adapters/GeneralAdapter1.sol:GeneralAdapter1 \
  --constructor-args "$BUNDLER3_ADDRESS" "$MORPHO_ADDRESS" "$WNATIVE" \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --verify --etherscan-api-key "$BASESCAN_API_KEY"
```

**Update .env**: `GENERAL_ADAPTER_ADDRESS=0x_PASTE_HERE`

---

### Step 6: `PUBLIC_ALLOCATOR_ADDRESS`

**What**: Allows anyone to reallocate liquidity between Morpho markets within vault-defined flow caps. Used by MetaMorpho/VaultV2 vaults.

**Repo**: `public-allocator`
**Contract**: `src/PublicAllocator.sol`
**Constructor args**: `address morpho` → `MORPHO_ADDRESS` from Step 1
**Depends on**: `MORPHO_ADDRESS`

**Deploy**:
```bash
cd ~/morpho/public-allocator
source ~/morpho/.env

forge create src/PublicAllocator.sol:PublicAllocator \
  --constructor-args "$MORPHO_ADDRESS" \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --verify --etherscan-api-key "$BASESCAN_API_KEY"
```

**Verify**:
```bash
# Should return your MORPHO_ADDRESS
cast call "$PUBLIC_ALLOCATOR_ADDRESS" "MORPHO()(address)" --rpc-url "$BASE_RPC_URL"
```

**Update .env**: `PUBLIC_ALLOCATOR_ADDRESS=0x_PASTE_HERE`

---

### Step 7: `PRE_LIQUIDATION_FACTORY_ADDRESS`

**What**: Factory that creates PreLiquidation contracts — these allow partial liquidations before a position hits the standard liquidation threshold, using linear incentive curves.

**Repo**: `pre-liquidation`
**Contract**: `src/PreLiquidationFactory.sol`
**Constructor args**: `address morpho` → `MORPHO_ADDRESS` from Step 1
**Depends on**: `MORPHO_ADDRESS`

**Deploy**:
```bash
cd ~/morpho/pre-liquidation
source ~/morpho/.env

forge create src/PreLiquidationFactory.sol:PreLiquidationFactory \
  --constructor-args "$MORPHO_ADDRESS" \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --verify --etherscan-api-key "$BASESCAN_API_KEY"
```

**Verify**:
```bash
# Should return your MORPHO_ADDRESS
cast call "$PRE_LIQUIDATION_FACTORY_ADDRESS" "MORPHO()(address)" --rpc-url "$BASE_RPC_URL"
```

**Update .env**: `PRE_LIQUIDATION_FACTORY_ADDRESS=0x_PASTE_HERE`

---

## Post-Deployment Checklist

After all 7 steps, run this verification script:

```bash
source ~/morpho/.env

echo "=== Morpho Blue Stack Verification ==="

echo -n "Morpho owner:          "; cast call "$MORPHO_ADDRESS" "owner()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "IRM enabled:           "; cast call "$MORPHO_ADDRESS" "isIrmEnabled(address)(bool)" "$IRM_ADDRESS" --rpc-url "$BASE_RPC_URL"
echo -n "Fee recipient:         "; cast call "$MORPHO_ADDRESS" "feeRecipient()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "IRM → Morpho:          "; cast call "$IRM_ADDRESS" "MORPHO()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "PublicAlloc → Morpho:  "; cast call "$PUBLIC_ALLOCATOR_ADDRESS" "MORPHO()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "PreLiq → Morpho:       "; cast call "$PRE_LIQUIDATION_FACTORY_ADDRESS" "MORPHO()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "Adapter → Morpho:      "; cast call "$GENERAL_ADAPTER_ADDRESS" "MORPHO()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "Adapter → Bundler:     "; cast call "$GENERAL_ADAPTER_ADDRESS" "BUNDLER3()(address)" --rpc-url "$BASE_RPC_URL"

echo ""
echo "All addresses should match. If any return address(0) or wrong values, redeploy that component."
```

**Expected output**: Every `→ Morpho` line should return the same `MORPHO_ADDRESS`. The adapter's `→ Bundler` should return `BUNDLER3_ADDRESS`.

---

## Dependency Graph (Quick Reference)

```
NOTHING NEEDED          NEEDS MORPHO_ADDRESS         NEEDS BUNDLER3 + MORPHO
─────────────────       ────────────────────         ───────────────────────
Morpho.sol              AdaptiveCurveIrm             GeneralAdapter1
Bundler3.sol            PublicAllocator
OracleFactory           PreLiquidationFactory
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `NOT_OWNER` on enableIrm | Caller is not Morpho owner | Use the same `DEPLOYER_PRIVATE_KEY` that deployed Morpho |
| `IRM_NOT_ENABLED` on createMarket | IRM not enabled on Morpho | Run `cast send` to call `enableIrm(address)` |
| `LLTV_NOT_ENABLED` on createMarket | LLTV value not enabled | Run `enableLltv(uint256)` on Morpho first |
| Contract verification fails | Wrong API key or BaseScan delay | Re-verify: `forge verify-contract $ADDRESS Contract --chain base` |
| Transaction reverts with no message | Insufficient ETH for gas | Check balance: `cast balance $DEPLOYER_ADDRESS --rpc-url $BASE_RPC_URL --ether` |
| Oracle returns 0 | Chainlink feed address wrong | Verify feed addresses at data.chain.link for Base |