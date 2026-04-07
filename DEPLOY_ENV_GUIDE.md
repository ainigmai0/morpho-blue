# Morpho Blue Full Lending Stack — Environment Variables Guide

> For DevOps deploying the complete Morpho lending stack to Base chain.
> Each variable is explained: what it is, where it comes from, and the exact command to obtain it.

---

## Prerequisites

Before starting, you need these set up independently:

| What | How to Get |
|------|------------|
| **Foundry** | `curl -L https://foundry.paradigm.xyz \| bash && foundryup` |
| **Funded wallet on Base** | ~0.1 ETH on Base (bridge via bridge.base.org) |
| **BaseScan API key** | Register at basescan.org → My Account → API Keys |

> **EVM Compatibility**: Base supports the Cancun (Dencun) upgrade. Bundler3 (`0.8.28`), PreLiquidation (`0.8.27`), and VaultV2 (`0.8.28`) require Cancun EVM for transient storage (EIP-1153). Morpho Blue core (`0.8.19`), IRM (`0.8.19`), Oracles (`0.8.21`), and PublicAllocator (`0.8.24`) target the Paris EVM.

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

# ═══════════════════════════════════════════════════════════════
# TIER 3 — Vault V2 (factories, roles, vault instances)
# ═══════════════════════════════════════════════════════════════
VAULT_V2_FACTORY=
MORPHO_VAULT_V1_ADAPTER_FACTORY=
MORPHO_MARKET_V1_ADAPTER_V2_FACTORY=
ADAPTER_REGISTRY=
OWNER=
CURATOR=
ALLOCATOR=
SENTINEL=
ASSET=
TIMELOCK_DURATION=
VAULT_TIMELOCK_DURATION=
ADAPTER_TIMELOCK_DURATION=
DEAD_DEPOSIT_AMOUNT=
MARKET_ID=
COLLATERAL_TOKEN_CAP=
MARKET_CAP=
VAULT_V1=
VAULT_V2_ADDRESS=
ADAPTER_ADDRESS=
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
- Leave unset — `feeRecipient` defaults to `address(0)` (no fees collected). No need to call `setFeeRecipient` if you don't want fees.

> **Note**: `setFeeRecipient` reverts with `ALREADY_SET` if the new address equals the current one. Since the default is `address(0)`, calling `setFeeRecipient(address(0))` will revert. Fees are also per-market — you must call `setFee(marketParams, fee)` on each market separately after setting a recipient.

**Set on-chain after Morpho deploys** (Step 1 below):
```bash
cast send "$MORPHO_ADDRESS" "setFeeRecipient(address)" "$FEE_RECIPIENT" \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY"
```

---

## Tier 2: Variables Produced by Deployment (Sequential)

Each step deploys a contract. The output address becomes an env var used by later steps. **You must follow this order.**

```
Step 1 ─── Morpho.sol(owner) ──────────────→ MORPHO_ADDRESS
              │    then: enableLltv(uint256) for each LLTV tier
              │
Step 2 ───── AdaptiveCurveIrm(morpho) ────→ IRM_ADDRESS
              │    then: morpho.enableIrm(irm)
              │
Step 3 ───── OracleFactory() ─────────────→ ORACLE_FACTORY_ADDRESS (optional)
              │    └── createOracle(...) ──→ ORACLE_WETH_USDC_ADDRESS
              │    └── createOracle(...) ──→ ORACLE_CBETH_USDC_ADDRESS
              │
Step 4 ───── Bundler3() ──────────────────→ BUNDLER3_ADDRESS
              │
Step 5 ───── GeneralAdapter1(bundler,morpho,weth) → GENERAL_ADAPTER_ADDRESS
              │
Step 6 ───── PublicAllocator(morpho) ─────→ PUBLIC_ALLOCATOR_ADDRESS
              │
Step 7 ───── PreLiquidationFactory(morpho) → PRE_LIQUIDATION_FACTORY_ADDRESS
```

---

### Step 1: `MORPHO_ADDRESS`

**What**: The Morpho Blue singleton — the core lending protocol. Everything else plugs into this.

**Repo**: `morpho-blue`
**Contract**: `src/Morpho.sol` (Solidity `0.8.19`, EVM: paris)
**Constructor args**: `address newOwner` — must be non-zero

> **Note**: The `morpho-blue` repo has no deploy scripts. Use `forge create` directly.

**Deploy**:
```bash
cd ~/morpho/morpho-blue
source ~/morpho/.env

forge create src/Morpho.sol:Morpho \
  --constructor-args "$DEPLOYER_ADDRESS" \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --verify --etherscan-api-key "$BASESCAN_API_KEY"
```

**Get the address**: Read `Deployed to:` from the output.

**Verify**:
```bash
# Should return your DEPLOYER_ADDRESS
cast call "$MORPHO_ADDRESS" "owner()(address)" --rpc-url "$BASE_RPC_URL"
```

**Post-deploy — enable LLTV values** (required before any market can be created):
```bash
# Enable common LLTV tiers (e.g., 86%, 91.5%, 94.5%, 77%, 62.5%)
cast send "$MORPHO_ADDRESS" "enableLltv(uint256)" 860000000000000000 \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY"
cast send "$MORPHO_ADDRESS" "enableLltv(uint256)" 915000000000000000 \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY"
```

> LLTV is scaled to 1e18 (WAD). 86% = `860000000000000000`. Must be < 1e18. Each value must be enabled individually. Market creation reverts with `LLTV_NOT_ENABLED` if the LLTV hasn't been enabled.

**Update .env**: `MORPHO_ADDRESS=0x_PASTE_HERE`

---

### Step 2: `IRM_ADDRESS`

**What**: The Adaptive Curve Interest Rate Model. Calculates borrow/supply rates for every market based on utilization. Parameters are hardcoded in `ConstantsLib.sol` (90% target utilization, 4% initial rate, 4x curve steepness).

**Repo**: `morpho-blue-irm`
**Contract**: `src/adaptive-curve-irm/AdaptiveCurveIrm.sol` (Solidity `0.8.19`, EVM: paris)
**Constructor args**: `address morpho` → use `MORPHO_ADDRESS` from Step 1
**Depends on**: `MORPHO_ADDRESS`

> **Note**: The `morpho-blue-irm` repo has no deploy scripts. Use `forge create` directly.

**Deploy**:
```bash
cd ~/morpho/morpho-blue-irm
source ~/morpho/.env

forge create src/adaptive-curve-irm/AdaptiveCurveIrm.sol:AdaptiveCurveIrm \
  --constructor-args "$MORPHO_ADDRESS" \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --verify --etherscan-api-key "$BASESCAN_API_KEY"
```

**Get the address**: Read `Deployed to:` from the output.

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
**Contract**: `src/morpho-chainlink/MorphoChainlinkOracleV2Factory.sol` (Solidity `0.8.21`, EVM: paris)
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
**Contract**: `src/Bundler3.sol` (Solidity `0.8.28`, EVM: cancun)
**Constructor args**: None (stateless, uses EIP-1153 transient storage)
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
**Contract**: `src/adapters/GeneralAdapter1.sol` (Solidity `0.8.28`, EVM: cancun)
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
**Contract**: `src/PublicAllocator.sol` (Solidity `0.8.24`, EVM: paris)
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
**Contract**: `src/PreLiquidationFactory.sol` (Solidity `0.8.27`, EVM: cancun)
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

## Tier 3: Deploy Vault V2 (ERC4626 Yield Vaults)

Vault V2 is the yield layer on top of Morpho Blue. It lets curators manage lending strategies through an ERC4626-compliant vault that users deposit into. This tier has the most env vars because vaults are configurable with roles, timelocks, and market adapters.

**Repo**: `vault-v2-deployment`
**Depends on**: All Tier 2 addresses must be populated first.

There are **two deployment scripts** — pick the one that matches your use case:

| Script | Use Case |
|--------|----------|
| `DeployVaultV2.s.sol` | Migrate from an existing MetaMorpho V1 vault |
| `DeployVaultV2WithMarketAdapter.s.sol` | Fresh vault with direct Morpho Blue market access (no V1 needed) |

### Tier 3 .env additions

Add these to your `~/morpho/.env`:

```bash
# ═══════════════════════════════════════════════════════════════
# TIER 3 — Vault V2 Infrastructure (deploy factories first)
# ═══════════════════════════════════════════════════════════════
VAULT_V2_FACTORY=
MORPHO_VAULT_V1_ADAPTER_FACTORY=
MORPHO_MARKET_V1_ADAPTER_V2_FACTORY=
ADAPTER_REGISTRY=

# ═══════════════════════════════════════════════════════════════
# TIER 3 — Vault Configuration
# ═══════════════════════════════════════════════════════════════
OWNER=
CURATOR=
ALLOCATOR=
SENTINEL=
ASSET=

# ═══════════════════════════════════════════════════════════════
# TIER 3 — Optional Timelocks & Caps
# ═══════════════════════════════════════════════════════════════
TIMELOCK_DURATION=
VAULT_TIMELOCK_DURATION=
ADAPTER_TIMELOCK_DURATION=
DEAD_DEPOSIT_AMOUNT=

# ═══════════════════════════════════════════════════════════════
# TIER 3 — Optional Market Config (WithMarketAdapter only)
# ═══════════════════════════════════════════════════════════════
MARKET_ID=
COLLATERAL_TOKEN_CAP=
MARKET_CAP=

# ═══════════════════════════════════════════════════════════════
# TIER 3 — Optional V1 Migration (DeployVaultV2 only)
# ═══════════════════════════════════════════════════════════════
VAULT_V1=

# ═══════════════════════════════════════════════════════════════
# TIER 3 — Post-Deploy (filled after vault deploys)
# ═══════════════════════════════════════════════════════════════
VAULT_V2_ADDRESS=
ADAPTER_ADDRESS=
```

---

### Step 8: Deploy Vault V2 Factories

Before deploying any vault, you need three factory contracts. These are deployed once and reused for every vault. All are in the `vault-v2` repo (Solidity `0.8.28`, EVM: cancun).

**Constructor args per factory**:
- `VaultV2Factory` — no args
- `MorphoVaultV1AdapterFactory` — no args
- `MorphoMarketV1AdapterV2Factory(address _morpho, address _adaptiveCurveIrm)` — needs both addresses

**Option A: Use the convenience script** (located in `test/script/`, review before mainnet use):

```bash
cd ~/morpho/vault-v2-deployment
source ~/morpho/.env

# DeployFactories uses MORPHO and ADAPTIVE_CURVE_IRM env var names (not MORPHO_ADDRESS/IRM_ADDRESS)
export MORPHO="$MORPHO_ADDRESS"
export ADAPTIVE_CURVE_IRM="$IRM_ADDRESS"

forge script test/script/DeployFactories.s.sol:DeployFactories \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast --verify \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  -vvv
```

> **Warning**: `DeployFactories.s.sol` is in `test/script/` and uses `vm.envOr` with `makeAddr()` fallbacks. If `MORPHO` or `ADAPTIVE_CURVE_IRM` are not set, it silently uses mock addresses — always verify these are set before running on mainnet.

**Option B: Deploy individually with `forge create`**:

```bash
cd ~/morpho/vault-v2
source ~/morpho/.env

# 1. VaultV2Factory (no args)
forge create src/VaultV2Factory.sol:VaultV2Factory \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY" \
  --verify --etherscan-api-key "$BASESCAN_API_KEY"

# 2. MorphoVaultV1AdapterFactory (no args)
forge create src/adapters/MorphoVaultV1AdapterFactory.sol:MorphoVaultV1AdapterFactory \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY" \
  --verify --etherscan-api-key "$BASESCAN_API_KEY"

# 3. MorphoMarketV1AdapterV2Factory (needs morpho + irm)
forge create src/adapters/MorphoMarketV1AdapterV2Factory.sol:MorphoMarketV1AdapterV2Factory \
  --constructor-args "$MORPHO_ADDRESS" "$IRM_ADDRESS" \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY" \
  --verify --etherscan-api-key "$BASESCAN_API_KEY"
```

**Console output** (3 addresses):
```
VaultV2Factory                    → Deployed to: 0x... ← VAULT_V2_FACTORY
MorphoVaultV1AdapterFactory       → Deployed to: 0x... ← MORPHO_VAULT_V1_ADAPTER_FACTORY
MorphoMarketV1AdapterV2Factory    → Deployed to: 0x... ← MORPHO_MARKET_V1_ADAPTER_V2_FACTORY
```

**Update .env** with all three addresses.

---

### Step 9: `ADAPTER_REGISTRY`

**What**: A contract implementing `IAdapterRegistry` — validates which adapters a vault can use. It has a single function: `isInRegistry(address) → bool`.

**How to obtain**:
- If Morpho has deployed a canonical registry on your chain, use that address (check [docs.morpho.org/resources/addresses](https://docs.morpho.org/get-started/resources/addresses/))
- If none exists, you can set `ADAPTER_REGISTRY` to `address(0)` — this **disables registry validation** and allows any adapter (acceptable for testing, not recommended for production)
- Or deploy your own contract implementing `IAdapterRegistry` (single function: `isInRegistry(address) → bool`)

> **Warning**: Both deploy scripts call `abdicate(setAdapterRegistry.selector)` during deployment. This permanently locks the registry address — it can never be changed after deployment. Choose carefully.

**Update .env**: `ADAPTER_REGISTRY=0x_PASTE_HERE` (or `0x0000000000000000000000000000000000000000` to skip)

---

### Step 10: Role Addresses (You Choose These)

These are not deployed — they are addresses you assign to control the vault.

#### `OWNER` (required)

**What**: Final owner of the deployed vault. Has full administrative control — can change all roles, add/remove adapters, set caps.

**How to get**: Your choice. Use your deployer address for testing, or a multisig (e.g., Safe) for production.

```bash
# Simplest: use deployer
OWNER=$DEPLOYER_ADDRESS

# Production: use a multisig
OWNER=0x_YOUR_SAFE_MULTISIG_ADDRESS
```

#### `CURATOR` (optional, defaults to OWNER)

**What**: Can manage vault parameters — set caps, configure markets, add adapters. Cannot transfer ownership.

**How to get**: Your choice. Set to a team member or operations address. If omitted in the `WithMarketAdapter` script, defaults to `OWNER`.

> **Note**: In `DeployVaultV2.s.sol` (V1 migration), `CURATOR` is **required** (`vm.envAddress`, not `vm.envOr`). You must set it explicitly.

#### `ALLOCATOR` (optional, defaults to OWNER)

**What**: Can allocate and deallocate funds between adapters/markets. This is the role that moves capital around.

**How to get**: Your choice. Typically a bot address or operations wallet. If omitted in the `WithMarketAdapter` script, defaults to `OWNER`.

> **Note**: In `DeployVaultV2.s.sol` (V1 migration), `ALLOCATOR` is **required**.

#### `SENTINEL` (optional, defaults to address(0))

**What**: Emergency guardian role. Can perform protective actions (e.g., force-deallocate in emergency). Set to `address(0)` to disable.

**How to get**: Your choice. Typically a security multisig or monitoring bot.

---

### Step 11: `ASSET`

**What**: The underlying ERC20 token that the vault accepts for deposits. This determines what users deposit and what the vault lends on Morpho Blue markets.

**How to get**: Use the token address on your target chain.

**Common Base mainnet tokens**:

| Token | Address | Decimals |
|-------|---------|----------|
| USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | 6 |
| WETH | `0x4200000000000000000000000000000000000006` | 18 |
| cbETH | `0x2Ae3F1Ec7F1F5012CFEab0185bfc7aa3cf0DEc22` | 18 |
| DAI | `0x50c5725949A6F0c72E6C4a641F24049A917DB0Cb` | 18 |

**Only used by**: `DeployVaultV2WithMarketAdapter.s.sol`
(The V1 migration script reads the asset from the source V1 vault automatically.)

---

### Step 12: Optional Timelock Configuration

Timelocks add a delay before sensitive vault changes take effect. **Required for Morpho listing** (minimum 3 days = 259200 seconds).

#### `TIMELOCK_DURATION` (DeployVaultV2 script)

**What**: Timelock for vault functions in the V1 migration script.
**Default**: `0` (no timelock)
**Listing requirement**: `259200` (3 days)

#### `VAULT_TIMELOCK_DURATION` (WithMarketAdapter script)

**What**: Timelock for vault-level functions (add adapter, set caps, etc.).
**Default**: `0`
**Listing requirement**: `259200`

#### `ADAPTER_TIMELOCK_DURATION` (WithMarketAdapter script)

**What**: Independent timelock on the MorphoMarketV1AdapterV2. Controls delay for adapter-specific changes.
**Default**: `0`
**Listing requirement**: `259200`

```bash
# For Morpho listing compliance:
VAULT_TIMELOCK_DURATION=259200
ADAPTER_TIMELOCK_DURATION=259200
```

---

### Step 13: Optional Dead Deposit

#### `DEAD_DEPOSIT_AMOUNT`

**What**: An initial deposit sent to `address(0xdead)` to prevent share inflation attacks (ERC4626 first-depositor attack).

**Defaults**:
- `DeployVaultV2.s.sol`: `0` if not set (skipped). Read via `vm.envExists`/`vm.envUint`.
- `DeployVaultV2WithMarketAdapter.s.sol`: Auto-calculated from asset decimals:
  - `1e9` for tokens with >= 10 decimals (e.g., WETH with 18 decimals)
  - `1e12` for tokens with <= 9 decimals (e.g., USDC with 6 decimals)
  - The `DEAD_DEPOSIT_AMOUNT` env var is **not read** by this script — it always auto-calculates.

**Important**: Your deployer wallet must hold enough of the `ASSET` token to cover this amount.

---

### Step 14: Optional Market Configuration (WithMarketAdapter only)

These configure the first Morpho Blue market the vault will lend to. Set `MARKET_ID` to `bytes32(0)` (or omit) to skip and configure markets later.

#### `MARKET_ID`

**What**: The bytes32 identifier of a Morpho Blue market. Computed from `keccak256(abi.encode(loanToken, collateralToken, oracle, irm, lltv))`.

**How to get**: After creating a market on Morpho, compute the ID:
```bash
cast keccak256 $(cast abi-encode "f(address,address,address,address,uint256)" \
  "$LOAN_TOKEN" "$COLLATERAL_TOKEN" "$ORACLE_ADDRESS" "$IRM_ADDRESS" "$LLTV")
```

Or read it from the `CreateMarket` event after calling `morpho.createMarket(...)`.

#### `COLLATERAL_TOKEN_CAP`

**What**: Maximum amount the adapter can allocate to markets using this collateral token. Set to `0` for unlimited.

#### `MARKET_CAP`

**What**: Maximum amount the adapter can allocate to this specific market. Set to `0` for unlimited.

---

### Step 15: V1 Migration Only (DeployVaultV2 script)

Skip this section if using `DeployVaultV2WithMarketAdapter`.

#### `VAULT_V1`

**What**: Address of the existing MetaMorpho V1 vault to migrate from. The script reads the vault's asset automatically and creates a V1 adapter pointing to it.

**How to get**: The address of your deployed MetaMorpho (ERC4626) vault. Read via `vm.envAddress("VAULT_V1")` — **required**, will revert if not set.

---

### Deploy the Vault

#### Option A: Fresh vault with direct market access (recommended for new deployments)

```bash
cd ~/morpho/vault-v2-deployment
source ~/morpho/.env

forge script script/DeployVaultV2WithMarketAdapter.s.sol:DeployVaultV2WithMarketAdapter \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast --verify \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  -vvv
```

#### Option B: V1 migration vault

```bash
cd ~/morpho/vault-v2-deployment
source ~/morpho/.env

forge script script/DeployVaultV2.s.sol:DeployVaultV2 \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast --verify \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  -vvv
```

**Console output**:
```
VaultV2 deployed at:              0x... ← VAULT_V2_ADDRESS
MorphoMarketV1AdapterV2 deployed: 0x... ← ADAPTER_ADDRESS (WithMarketAdapter)
MorphoVaultV1Adapter deployed:    0x... ← ADAPTER_ADDRESS (V1 migration)
```

**Update .env**: `VAULT_V2_ADDRESS=0x...`, `ADAPTER_ADDRESS=0x...`

---

### Tier 3 Variable Summary

| Variable | Required | Source | Script |
|----------|----------|--------|--------|
| `VAULT_V2_FACTORY` | Yes | Step 8: deploy `VaultV2Factory` (no constructor args) | Both |
| `MORPHO_VAULT_V1_ADAPTER_FACTORY` | Yes | Step 8: deploy `MorphoVaultV1AdapterFactory` (no constructor args) | DeployVaultV2 |
| `MORPHO_MARKET_V1_ADAPTER_V2_FACTORY` | Yes | Step 8: deploy `MorphoMarketV1AdapterV2Factory(morpho, irm)` | WithMarketAdapter |
| `ADAPTER_REGISTRY` | Yes | Morpho canonical or `address(0)` | Both |
| `OWNER` | Yes | You choose (wallet or multisig) | Both |
| `CURATOR` | **Yes** (V1) / Optional (Market) | You choose — `vm.envAddress` in V1, `vm.envOr` in Market (defaults to OWNER) | Both |
| `ALLOCATOR` | **Yes** (V1) / Optional (Market) | You choose — `vm.envAddress` in V1, `vm.envOr` in Market (defaults to OWNER) | Both |
| `SENTINEL` | No | You choose (defaults to address(0)) | Both |
| `ASSET` | Yes | Token address on target chain | WithMarketAdapter only |
| `VAULT_V1` | Yes | Existing MetaMorpho V1 address | DeployVaultV2 only |
| `TIMELOCK_DURATION` | No | You choose (259200 for listing) | DeployVaultV2 only |
| `VAULT_TIMELOCK_DURATION` | No | You choose (259200 for listing) | WithMarketAdapter only |
| `ADAPTER_TIMELOCK_DURATION` | No | You choose (259200 for listing) | WithMarketAdapter only |
| `DEAD_DEPOSIT_AMOUNT` | No | 0 if not set (V1); ignored by Market (auto-calculated) | DeployVaultV2 only |
| `MARKET_ID` | No | Computed from market params (bytes32(0) to skip) | WithMarketAdapter only |
| `COLLATERAL_TOKEN_CAP` | No | You choose (0 = unlimited) | WithMarketAdapter only |
| `MARKET_CAP` | No | You choose (0 = unlimited) | WithMarketAdapter only |

---

### Tier 3 Dependency Chain

```
Tier 2 complete (MORPHO_ADDRESS + IRM_ADDRESS required)
      │
Step 8:  Deploy Factories
      │     ├── VaultV2Factory()                              → VAULT_V2_FACTORY
      │     ├── MorphoVaultV1AdapterFactory()                 → MORPHO_VAULT_V1_ADAPTER_FACTORY
      │     └── MorphoMarketV1AdapterV2Factory(morpho, irm)   → MORPHO_MARKET_V1_ADAPTER_V2_FACTORY
      │
Step 9:  ADAPTER_REGISTRY (canonical or address(0))
      │
Steps 10-14: Configure roles, asset, timelocks, caps (your choices)
      │
Step 15: Deploy vault (pick one script)
      │     ├── VAULT_V2_ADDRESS
      │     └── ADAPTER_ADDRESS
```

> **IMPORTANT**: The `vault-v2-deployment` repo is marked as educational/demonstration and is not audited. Review scripts thoroughly before mainnet use.

---

## Post-Deployment Checklist

After all steps (Tier 2 + Tier 3), run this verification script:

```bash
source ~/morpho/.env

echo "=== Tier 2: Core Stack Verification ==="

echo -n "Morpho owner:          "; cast call "$MORPHO_ADDRESS" "owner()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "IRM enabled:           "; cast call "$MORPHO_ADDRESS" "isIrmEnabled(address)(bool)" "$IRM_ADDRESS" --rpc-url "$BASE_RPC_URL"
echo -n "Fee recipient:         "; cast call "$MORPHO_ADDRESS" "feeRecipient()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "IRM → Morpho:          "; cast call "$IRM_ADDRESS" "MORPHO()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "PublicAlloc → Morpho:  "; cast call "$PUBLIC_ALLOCATOR_ADDRESS" "MORPHO()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "PreLiq → Morpho:       "; cast call "$PRE_LIQUIDATION_FACTORY_ADDRESS" "MORPHO()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "Adapter → Morpho:      "; cast call "$GENERAL_ADAPTER_ADDRESS" "MORPHO()(address)" --rpc-url "$BASE_RPC_URL"
echo -n "Adapter → Bundler:     "; cast call "$GENERAL_ADAPTER_ADDRESS" "BUNDLER3()(address)" --rpc-url "$BASE_RPC_URL"

echo ""
echo "=== Tier 3: Vault V2 Verification ==="

if [ -n "$VAULT_V2_ADDRESS" ]; then
  echo -n "Vault owner:           "; cast call "$VAULT_V2_ADDRESS" "owner()(address)" --rpc-url "$BASE_RPC_URL"
  echo -n "Vault curator:         "; cast call "$VAULT_V2_ADDRESS" "curator()(address)" --rpc-url "$BASE_RPC_URL"
  echo -n "Vault asset:           "; cast call "$VAULT_V2_ADDRESS" "asset()(address)" --rpc-url "$BASE_RPC_URL"
  echo -n "Vault registry:        "; cast call "$VAULT_V2_ADDRESS" "adapterRegistry()(address)" --rpc-url "$BASE_RPC_URL"
else
  echo "VAULT_V2_ADDRESS not set — skipping Tier 3 checks"
fi

echo ""
echo "All '→ Morpho' lines should return the same MORPHO_ADDRESS."
echo "Vault owner should match OWNER. Vault asset should match ASSET."
```

**Expected output**: Every `→ Morpho` line returns `MORPHO_ADDRESS`. Vault owner matches `OWNER`. Vault asset matches `ASSET`.

---

## Dependency Graph (Quick Reference)

```
TIER 2:
NO DEPENDENCIES         NEEDS MORPHO_ADDRESS         NEEDS BUNDLER3 + MORPHO
─────────────────       ────────────────────         ───────────────────────
Morpho.sol              AdaptiveCurveIrm             GeneralAdapter1
  + enableLltv()        PublicAllocator
  + enableIrm()         PreLiquidationFactory
Bundler3.sol
OracleFactory

TIER 3:
NO DEPENDENCIES                NEEDS MORPHO + IRM                YOUR CHOICES
──────────────────────         ──────────────────────            ──────────────────
VaultV2Factory (no args)       MorphoMarketV1AdapterV2Factory   OWNER (required)
MorphoVaultV1AdapterFactory      (morpho, irm)                  CURATOR, ALLOCATOR
  (no args)                    ADAPTER_REGISTRY                  SENTINEL, ASSET
                                 (canonical/address(0))          Timelocks, Caps
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
| `NotInAdapterRegistry()` | Adapter not in registry | Set `ADAPTER_REGISTRY=address(0)` to disable, or register adapter |
| Vault deploy reverts on dead deposit | Deployer lacks ASSET tokens | Fund deployer with enough ASSET tokens for `DEAD_DEPOSIT_AMOUNT` |
| Vault name rejected for listing | Name contains "morpho" | Vault name/symbol cannot contain "morpho" (case insensitive) |
| Timelock too short for listing | Duration < 259200 | Set `VAULT_TIMELOCK_DURATION=259200` and `ADAPTER_TIMELOCK_DURATION=259200` |