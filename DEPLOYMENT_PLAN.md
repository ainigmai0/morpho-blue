# Morpho Blue Full Stack — VPS Deployment Plan (Base Chain)

## Overview

Deploy the **complete Morpho lending stack** to **Base mainnet** (Chain ID: 8453) from a VPS. This covers all 12 repos — smart contracts, infrastructure, indexer, and frontend.

```
┌─────────────────────────────────────────────────────────────────┐
│                        USERS / FRONTEND                         │
│                     earn-basic-app (React)                       │
│                   static deploy (Vercel/Nginx)                   │
└───────────┬─────────────────────────────────────────────────────┘
            │ GraphQL (Apollo)              │ Viem/Wagmi (via SDKs)
            ▼                               ▼
┌───────────────────┐          ┌─────────────────────┐
│  morpho-blue-     │          │       eRPC           │
│  subgraph         │          │  fault-tolerant RPC   │
│  (TheGraph)       │          │  proxy (port 4000)    │
└───────────────────┘          └──────────┬───────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                      BASE MAINNET (Chain 8453)                   │
│                                                                  │
│  ┌──────────┐  ┌───────────┐  ┌────────────┐  ┌─────────────┐  │
│  │  Bundler3 │  │  Public   │  │    Pre-    │  │  Vault V2   │  │
│  │ +Adapters │  │ Allocator │  │Liquidation │  │  (ERC4626)  │  │
│  └─────┬────┘  └─────┬─────┘  └──────┬─────┘  └──────┬──────┘  │
│        │              │               │               │          │
│        │         ┌────▼───────────────▼───────────────▼┐         │
│        │         │                                      │         │
│        └────────►│           MORPHO BLUE               │         │
│                  │         (singleton)                   │         │
│                  │                                      │         │
│                  └──────┬────────────────┬──────────────┘         │
│                         │                │                        │
│                  ┌──────▼──────┐  ┌──────▼──────┐                │
│                  │  Chainlink  │  │  Adaptive   │                │
│                  │  Oracles    │  │  Curve IRM  │                │
│                  └─────────────┘  └─────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Repository Map

All repos are under the `morpho-org` GitHub organization (replace with your fork org as needed).

| # | Repo | Type | Solidity Version | Deploys To |
|---|------|------|-----------------|-----------|
| 1 | `morpho-blue` | Smart contract | `0.8.19` | Base chain |
| 2 | `morpho-blue-irm` | Smart contract | `0.8.19` | Base chain |
| 3 | `morpho-blue-oracles` | Smart contract | `0.8.21` | Base chain |
| 4 | `vault-v2` | Smart contract | `0.8.28` | Base chain |
| 5 | `vault-v2-deployment` | Deploy scripts (Foundry) | `0.8.28` | N/A (tooling) |
| 6 | `bundler3` | Smart contract | `0.8.28` | Base chain |
| 7 | `public-allocator` | Smart contract | `0.8.24` | Base chain |
| 8 | `pre-liquidation` | Smart contract | `0.8.27` | Base chain |
| 9 | `erpc` | Go service (erpc/erpc) | N/A | VPS (port 4000) |
| 10 | `morpho-blue-subgraph` | TheGraph indexer | N/A | VPS (Docker) |
| 11 | `sdks` | TypeScript lib (pnpm monorepo) | N/A | npm (build only) |
| 12 | `earn-basic-app` | React + Vite frontend | N/A | Static host |

> **Note on vault-v2-deployment**: This repo is marked as educational/demonstration and is not audited. Review scripts thoroughly before mainnet use.

---

## Phase 1: VPS Setup

### 1.1 Server Requirements

| Resource | Minimum | Notes |
|----------|---------|-------|
| **OS** | Ubuntu 22.04+ LTS | |
| **RAM** | 16 GB | Subgraph indexer + eRPC + builds |
| **Disk** | 100 GB SSD | Subgraph PostgreSQL + IPFS data grows over time |
| **CPU** | 4 cores | |
| **Network** | Stable outbound HTTPS | Inbound: 22 (SSH), 80/443 (Nginx). Do NOT expose 4000/8000 directly |

### 1.2 Install Tooling

```bash
# System packages
sudo apt update && sudo apt install -y git build-essential curl wget jq

# ── Foundry (forge, cast, anvil, chisel) ──
# Ref: https://book.getfoundry.sh/getting-started/installation
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc   # adds ~/.foundry/bin to PATH
foundryup          # downloads latest forge/cast/anvil/chisel

# Verify
forge --version    # e.g. forge 1.x.x
cast --version

# ── Node.js 22 LTS & pnpm ──
# Ref: https://nodejs.org/en/download/package-manager
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo bash -
sudo apt install -y nodejs
node --version     # v22.x.x

# pnpm (required by sdks and earn-basic-app)
npm install -g pnpm
pnpm --version

# yarn classic (required by morpho-blue-subgraph)
npm install -g yarn
yarn --version     # 1.22.x

# ── Go 1.25+ (required by eRPC — see erpc go.mod) ──
# Check https://go.dev/dl/ for the latest 1.25.x patch
GO_VERSION="1.25.0"
wget "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz"
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
go version         # go1.25.0 linux/amd64

# ── Docker & Docker Compose (for subgraph) ──
# Ref: https://docs.docker.com/engine/install/ubuntu/
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Log out and back in for group membership to take effect
newgrp docker
docker --version
docker compose version  # Docker Compose V2 (bundled with Docker)
```

### 1.3 Clone All Repos

```bash
mkdir -p ~/morpho && cd ~/morpho

# Replace "ainigmai0" with your GitHub org/user
GH_ORG="ainigmai0"

repos=(
  morpho-blue morpho-blue-irm morpho-blue-oracles
  vault-v2 vault-v2-deployment
  bundler3 public-allocator pre-liquidation
  sdks morpho-blue-subgraph earn-basic-app
)

for repo in "${repos[@]}"; do
  git clone "https://github.com/${GH_ORG}/${repo}.git"
done

# eRPC is from a different org
git clone "https://github.com/erpc/erpc.git"

# Initialize submodules and build Foundry projects
for repo in morpho-blue morpho-blue-irm morpho-blue-oracles vault-v2 bundler3 public-allocator pre-liquidation vault-v2-deployment; do
  echo "=== Building $repo ==="
  cd ~/morpho/$repo
  git submodule update --init --recursive
  forge build
  cd ~/morpho
done
```

### 1.4 Master Environment File

Create a shared `.env` file. All deploy scripts will `source` this.

```bash
cat > ~/morpho/.env << 'ENVEOF'
# ═══════════════════════════════════════════
# RPC
# ═══════════════════════════════════════════
# Public RPC (rate-limited, OK for deploy but not production indexing):
BASE_RPC_URL=https://mainnet.base.org

# Paid RPC (strongly recommended for subgraph indexing & reliability):
# BASE_RPC_URL=https://base-mainnet.g.alchemy.com/v2/YOUR_KEY
# or
# BASE_RPC_URL=https://base-mainnet.infura.io/v3/YOUR_KEY

# ═══════════════════════════════════════════
# WALLET
# ═══════════════════════════════════════════
DEPLOYER_PRIVATE_KEY=0x_REPLACE_WITH_ACTUAL_KEY
DEPLOYER_ADDRESS=0x_REPLACE_WITH_DEPLOYER_ADDRESS
BASESCAN_API_KEY=_REPLACE_WITH_BASESCAN_API_KEY

# ═══════════════════════════════════════════
# DEPLOYED ADDRESSES (fill after each phase)
# ═══════════════════════════════════════════
MORPHO_ADDRESS=
IRM_ADDRESS=
ORACLE_FACTORY_ADDRESS=
ORACLE_WETH_USDC_ADDRESS=
ORACLE_CBETH_USDC_ADDRESS=
BUNDLER3_ADDRESS=
GENERAL_ADAPTER_ADDRESS=
PUBLIC_ALLOCATOR_ADDRESS=
PRE_LIQUIDATION_FACTORY_ADDRESS=
VAULT_V2_FACTORY_ADDRESS=
VAULT_V2_ADDRESS=
ADAPTER_FACTORY_ADDRESS=
ADAPTER_ADDRESS=
FEE_RECIPIENT=

# Market IDs (bytes32, computed after market creation)
MARKET_ID_WETH_USDC=
MARKET_ID_CBETH_USDC=
ENVEOF
```

> **IMPORTANT**: After each deployment phase, update the addresses in `~/morpho/.env` before proceeding to the next phase. Many scripts depend on previously deployed addresses.

### 1.5 Fund Deployer Wallet

- Need approximately **0.1 ETH on Base** (entire stack deploys for < $5 at typical L2 gas prices)
- Bridge from Ethereum L1 via [bridge.base.org](https://bridge.base.org)
- Or purchase ETH directly on Base via an exchange that supports Base withdrawals

```bash
# Verify balance
cast balance $DEPLOYER_ADDRESS --rpc-url $BASE_RPC_URL --ether
```

---

## Phase 2: Base Chain Reference Addresses

### Tokens (Verified on BaseScan)

| Token | Address | Decimals | Notes |
|-------|---------|----------|-------|
| WETH  | `0x4200000000000000000000000000000000000006` | 18 | OP Stack pre-deploy (canonical) |
| USDC  | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | 6 | Native Circle USDC on Base |
| USDbC | `0xd9aAEc86B65D86f6A7B5B1b0c42FFA531710b6CA` | 6 | Bridged USDC (legacy, being deprecated) |
| DAI   | `0x50c5725949A6F0c72E6C4a641F24049A917DB0Cb` | 18 | OptimismMintableERC20 |
| cbETH | `0x2Ae3F1Ec7F1F5012CFEab0185bfc7aa3cf0DEc22` | 18 | Coinbase Wrapped Staked ETH |

> **Warning**: Use native USDC (`0x8335...`), NOT USDbC, for new deployments. USDbC is the old bridged version.

### Chainlink Price Feeds (Base Mainnet — Verified on data.chain.link)

| Feed | Address | Decimals | Source |
|------|---------|----------|--------|
| ETH/USD  | `0x71041dddad3595F9CEd3DcCFBe3D1F4b0a16Bb70` | 8 | [data.chain.link](https://data.chain.link/feeds/base/mainnet/eth-usd) |
| USDC/USD | `0x7e860098F58bBFC8648a4311b374B1D669a2bc6B` | 8 | [data.chain.link](https://data.chain.link/feeds/base/mainnet/usdc-usd) |
| cbETH/ETH | `0x806b4Ac04501c29769051e42783cF04dCE41440b` | 18 | [data.chain.link](https://data.chain.link/feeds/base/mainnet/cbeth-eth) |
| DAI/USD  | `0x591e79239a7d679378eC8c847e5038150364C78F` | 8 | [data.chain.link](https://data.chain.link/feeds/base/mainnet/dai-usd) |

> **Always verify** feed addresses at [docs.chain.link/data-feeds/price-feeds/addresses](https://docs.chain.link/data-feeds/price-feeds/addresses) before deploying. Chainlink may deprecate or migrate feeds.

---

## Phase 3: Deploy Morpho Blue Core

**Source**: `morpho-blue/src/Morpho.sol`
**Solidity**: `0.8.19` — Optimizer: 999,999 runs, EVM target: paris, via-ir: enabled
**Constructor**: `constructor(address newOwner)` — single arg, must be non-zero

### 3.1 Deploy Script

Create `morpho-blue/script/DeployMorpho.s.sol`:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity 0.8.19;

import "forge-std/Script.sol";
import "../src/Morpho.sol";

contract DeployMorpho is Script {
    function run() external {
        uint256 deployerKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
        address owner = vm.addr(deployerKey);

        vm.startBroadcast(deployerKey);
        Morpho morpho = new Morpho(owner);
        vm.stopBroadcast();

        console.log("Morpho Blue:", address(morpho));
    }
}
```

### 3.2 Deploy

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

**After deploy**: Copy the logged address and update `~/morpho/.env`:
```bash
# Edit ~/morpho/.env and set:
MORPHO_ADDRESS=0x_PASTE_ADDRESS_HERE
```

### 3.3 Verify Deployment

```bash
source ~/morpho/.env
# Should return the deployer address
cast call "$MORPHO_ADDRESS" "owner()(address)" --rpc-url "$BASE_RPC_URL"
```

---

## Phase 4: Deploy AdaptiveCurveIrm

**Source**: `morpho-blue-irm/src/adaptive-curve-irm/AdaptiveCurveIrm.sol`
**Solidity**: `0.8.19`
**Constructor**: `constructor(address morpho)` — takes the Morpho Blue address

**IRM Parameters** (hardcoded in `src/adaptive-curve-irm/libraries/ConstantsLib.sol` — not configurable at deploy time):

| Parameter | Value | Description |
|-----------|-------|-------------|
| `TARGET_UTILIZATION` | 90% (0.9e18) | Optimal utilization rate |
| `INITIAL_RATE_AT_TARGET` | 4% APR | Starting rate at target utilization |
| `MIN_RATE_AT_TARGET` | 0.1% APR | Floor (min borrow rate = 0.025%) |
| `MAX_RATE_AT_TARGET` | 200% APR | Ceiling (max borrow rate = 800%) |
| `CURVE_STEEPNESS` | 4 (4e18) | How sharply rates increase above target |
| `ADJUSTMENT_SPEED` | 50/year | How fast rates adapt to utilization changes |

### 4.1 Deploy Script

Create `morpho-blue-irm/script/DeployIrm.s.sol`:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity 0.8.19;

import "forge-std/Script.sol";
import "../src/adaptive-curve-irm/AdaptiveCurveIrm.sol";

contract DeployIrm is Script {
    function run() external {
        uint256 deployerKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
        address morpho = vm.envAddress("MORPHO_ADDRESS");

        vm.startBroadcast(deployerKey);
        AdaptiveCurveIrm irm = new AdaptiveCurveIrm(morpho);
        vm.stopBroadcast();

        console.log("AdaptiveCurveIrm:", address(irm));
    }
}
```

### 4.2 Deploy

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

**After deploy**: Update `~/morpho/.env` → `IRM_ADDRESS=0x...`

---

## Phase 5: Deploy Oracles

**Source**: `morpho-blue-oracles/src/morpho-chainlink/MorphoChainlinkOracleV2.sol`
**Solidity**: `0.8.21`
**Factory**: `MorphoChainlinkOracleV2Factory` — no constructor args (stateless)

**Factory `createMorphoChainlinkOracleV2` signature**:
```solidity
function createMorphoChainlinkOracleV2(
    IERC4626 baseVault,                      // Vault for collateral token (address(0) if none)
    uint256 baseVaultConversionSample,       // 1 if no vault
    AggregatorV3Interface baseFeed1,         // Primary price feed for collateral
    AggregatorV3Interface baseFeed2,         // Secondary feed (address(0) if unused)
    uint256 baseTokenDecimals,               // Collateral token decimals
    IERC4626 quoteVault,                     // Vault for loan token (address(0) if none)
    uint256 quoteVaultConversionSample,      // 1 if no vault
    AggregatorV3Interface quoteFeed1,        // Primary price feed for loan token
    AggregatorV3Interface quoteFeed2,        // Secondary feed (address(0) if unused)
    uint256 quoteTokenDecimals,              // Loan token decimals
    bytes32 salt                             // CREATE2 salt for deterministic address
) external returns (IMorphoChainlinkOracleV2)
```

### 5.1 Deploy Script

Create `morpho-blue-oracles/script/DeployOracles.s.sol`:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity 0.8.21;

import "forge-std/Script.sol";
import "../src/morpho-chainlink/MorphoChainlinkOracleV2.sol";
import "../src/morpho-chainlink/MorphoChainlinkOracleV2Factory.sol";

contract DeployOracles is Script {
    function run() external {
        uint256 deployerKey = vm.envUint("DEPLOYER_PRIVATE_KEY");

        vm.startBroadcast(deployerKey);

        // Factory (reusable, stateless)
        MorphoChainlinkOracleV2Factory factory = new MorphoChainlinkOracleV2Factory();
        console.log("Oracle Factory:", address(factory));

        // ── Oracle: WETH/USDC ──
        // Collateral: WETH (price via ETH/USD feed)
        // Loan: USDC (price via USDC/USD feed)
        IMorphoChainlinkOracleV2 oracleWethUsdc = factory.createMorphoChainlinkOracleV2(
            IERC4626(address(0)),                                                    // baseVault: none
            1,                                                                       // baseVaultConversionSample
            AggregatorV3Interface(0x71041dddad3595F9CEd3DcCFBe3D1F4b0a16Bb70),      // baseFeed1: ETH/USD
            AggregatorV3Interface(address(0)),                                       // baseFeed2: unused
            18,                                                                      // WETH decimals
            IERC4626(address(0)),                                                    // quoteVault: none
            1,                                                                       // quoteVaultConversionSample
            AggregatorV3Interface(0x7e860098F58bBFC8648a4311b374B1D669a2bc6B),       // quoteFeed1: USDC/USD
            AggregatorV3Interface(address(0)),                                       // quoteFeed2: unused
            6,                                                                       // USDC decimals
            bytes32(uint256(1))                                                      // salt
        );
        console.log("Oracle WETH/USDC:", address(oracleWethUsdc));

        // ── Oracle: cbETH/USDC ──
        // Collateral: cbETH (price via cbETH/ETH * ETH/USD — two feeds chained)
        // Loan: USDC (price via USDC/USD feed)
        IMorphoChainlinkOracleV2 oracleCbethUsdc = factory.createMorphoChainlinkOracleV2(
            IERC4626(address(0)),                                                    // baseVault: none
            1,                                                                       // baseVaultConversionSample
            AggregatorV3Interface(0x806b4Ac04501c29769051e42783cF04dCE41440b),      // baseFeed1: cbETH/ETH
            AggregatorV3Interface(0x71041dddad3595F9CEd3DcCFBe3D1F4b0a16Bb70),      // baseFeed2: ETH/USD
            18,                                                                      // cbETH decimals
            IERC4626(address(0)),                                                    // quoteVault: none
            1,                                                                       // quoteVaultConversionSample
            AggregatorV3Interface(0x7e860098F58bBFC8648a4311b374B1D669a2bc6B),       // quoteFeed1: USDC/USD
            AggregatorV3Interface(address(0)),                                       // quoteFeed2: unused
            6,                                                                       // USDC decimals
            bytes32(uint256(2))                                                      // salt
        );
        console.log("Oracle cbETH/USDC:", address(oracleCbethUsdc));

        vm.stopBroadcast();
    }
}
```

### 5.2 Deploy

```bash
cd ~/morpho/morpho-blue-oracles
source ~/morpho/.env

forge script script/DeployOracles.s.sol:DeployOracles \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast --verify \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  -vvv
```

### 5.3 Verify Oracle Prices

```bash
source ~/morpho/.env

# WETH/USDC oracle — returns price scaled to 36 + (loan_decimals - collateral_decimals) = 36 + (6 - 18) = 24 decimals
# At ETH = $3,000: expect ~3000 * 1e24 = 3000000000000000000000000000
cast call "$ORACLE_WETH_USDC_ADDRESS" "price()(uint256)" --rpc-url "$BASE_RPC_URL"

# cbETH/USDC oracle — similar scale
cast call "$ORACLE_CBETH_USDC_ADDRESS" "price()(uint256)" --rpc-url "$BASE_RPC_URL"
```

> **Sanity check**: Divide the returned value by 1e24 — you should get approximately the current ETH or cbETH price in USD.

**After deploy**: Update `~/morpho/.env` → `ORACLE_FACTORY_ADDRESS`, `ORACLE_WETH_USDC_ADDRESS`, `ORACLE_CBETH_USDC_ADDRESS`

---

## Phase 6: Create Morpho Blue Markets

This phase enables the IRM, sets LLTV thresholds, and creates lending markets on the Morpho singleton.

**Key functions** (from `IMorpho.sol`):
- `enableIrm(address irm)` — owner-only, whitelist an IRM contract
- `enableLltv(uint256 lltv)` — owner-only, whitelist an LLTV value
- `createMarket(MarketParams memory)` — permissionless, creates a new isolated market
- `setFee(MarketParams memory, uint256 newFee)` — owner-only, set market fee (max 25%)
- `setFeeRecipient(address)` — owner-only

**MarketParams struct**:
```solidity
struct MarketParams {
    address loanToken;        // Token being lent/borrowed
    address collateralToken;  // Token used as collateral
    address oracle;           // Chainlink oracle for price
    address irm;              // Interest rate model
    uint256 lltv;             // Liquidation LTV (e.g. 0.86e18 = 86%)
}
```

### 6.1 Deploy Script

Create `morpho-blue/script/SetupMarkets.s.sol`:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity 0.8.19;

import "forge-std/Script.sol";
import "../src/Morpho.sol";
import "../src/interfaces/IMorpho.sol";

contract SetupMarkets is Script {
    function run() external {
        uint256 deployerKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
        address morphoAddr = vm.envAddress("MORPHO_ADDRESS");
        address irmAddr = vm.envAddress("IRM_ADDRESS");
        address oracleWethUsdc = vm.envAddress("ORACLE_WETH_USDC_ADDRESS");
        address oracleCbethUsdc = vm.envAddress("ORACLE_CBETH_USDC_ADDRESS");
        address feeRecipient = vm.envAddress("FEE_RECIPIENT");

        // Base mainnet token addresses (verified on BaseScan)
        address USDC  = 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913;
        address WETH  = 0x4200000000000000000000000000000000000006;
        address cbETH = 0x2Ae3F1Ec7F1F5012CFEab0185bfc7aa3cf0DEc22;

        IMorpho morpho = IMorpho(morphoAddr);

        vm.startBroadcast(deployerKey);

        // ── Global config (owner-only) ──
        morpho.enableIrm(irmAddr);
        morpho.enableLltv(0.80e18);   // 80%
        morpho.enableLltv(0.86e18);   // 86%
        morpho.enableLltv(0.915e18);  // 91.5%
        morpho.setFeeRecipient(feeRecipient);

        // ── Market 1: WETH/USDC @ 86% LLTV ──
        MarketParams memory m1 = MarketParams({
            loanToken: USDC,
            collateralToken: WETH,
            oracle: oracleWethUsdc,
            irm: irmAddr,
            lltv: 0.86e18
        });
        morpho.createMarket(m1);
        morpho.setFee(m1, 0.10e18);  // 10% protocol fee on interest

        // ── Market 2: cbETH/USDC @ 80% LLTV ──
        MarketParams memory m2 = MarketParams({
            loanToken: USDC,
            collateralToken: cbETH,
            oracle: oracleCbethUsdc,
            irm: irmAddr,
            lltv: 0.80e18
        });
        morpho.createMarket(m2);
        morpho.setFee(m2, 0.10e18);  // 10% protocol fee on interest

        vm.stopBroadcast();

        console.log("Markets created successfully");
    }
}
```

### 6.2 Deploy

```bash
cd ~/morpho/morpho-blue
source ~/morpho/.env

forge script script/SetupMarkets.s.sol:SetupMarkets \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast \
  -vvv
```

### 6.3 Compute Market IDs

Market IDs are `keccak256(abi.encode(MarketParams))`. Compute them for later use:

```bash
source ~/morpho/.env

# Market ID for WETH/USDC @ 86% LLTV
MARKET_ID_WETH_USDC=$(cast keccak256 $(cast abi-encode \
  "x(address,address,address,address,uint256)" \
  "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913" \
  "0x4200000000000000000000000000000000000006" \
  "$ORACLE_WETH_USDC_ADDRESS" \
  "$IRM_ADDRESS" \
  "860000000000000000"))

echo "WETH/USDC Market ID: $MARKET_ID_WETH_USDC"

# Market ID for cbETH/USDC @ 80% LLTV
MARKET_ID_CBETH_USDC=$(cast keccak256 $(cast abi-encode \
  "x(address,address,address,address,uint256)" \
  "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913" \
  "0x2Ae3F1Ec7F1F5012CFEab0185bfc7aa3cf0DEc22" \
  "$ORACLE_CBETH_USDC_ADDRESS" \
  "$IRM_ADDRESS" \
  "800000000000000000"))

echo "cbETH/USDC Market ID: $MARKET_ID_CBETH_USDC"
```

**After deploy**: Update `~/morpho/.env` → `MARKET_ID_WETH_USDC`, `MARKET_ID_CBETH_USDC`

---

## Phase 7: Deploy Bundler3

Enables users to batch multiple operations atomically (approve + supply + borrow in 1 tx).

**Source**: `bundler3/src/Bundler3.sol` + `bundler3/src/adapters/GeneralAdapter1.sol`
**Solidity**: `0.8.28`

- `Bundler3` — **no constructor args** (uses transient storage)
- `GeneralAdapter1` — `constructor(address bundler3, address morpho, address wNative)`

### 7.1 Deploy Script

Create `bundler3/script/DeployBundler.s.sol`:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity ^0.8.28;

import "forge-std/Script.sol";
import "../src/Bundler3.sol";
import "../src/adapters/GeneralAdapter1.sol";

contract DeployBundler is Script {
    function run() external {
        uint256 deployerKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
        address morpho = vm.envAddress("MORPHO_ADDRESS");
        address wNative = 0x4200000000000000000000000000000000000006; // WETH on Base

        vm.startBroadcast(deployerKey);

        // Core bundler (no constructor args — uses transient storage)
        Bundler3 bundler = new Bundler3();
        console.log("Bundler3:", address(bundler));

        // General adapter (handles ERC20, ERC4626, Morpho, Permit2 operations)
        GeneralAdapter1 adapter = new GeneralAdapter1(
            address(bundler),
            morpho,
            wNative
        );
        console.log("GeneralAdapter1:", address(adapter));

        vm.stopBroadcast();
    }
}
```

### 7.2 Deploy

```bash
cd ~/morpho/bundler3
source ~/morpho/.env

forge script script/DeployBundler.s.sol:DeployBundler \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast --verify \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  -vvv
```

**After deploy**: Update `~/morpho/.env` → `BUNDLER3_ADDRESS`, `GENERAL_ADAPTER_ADDRESS`

---

## Phase 8: Deploy Public Allocator

Allows anyone to reallocate idle liquidity across Vault V2 markets without being the curator.

**Source**: `public-allocator/src/PublicAllocator.sol`
**Solidity**: `0.8.24`
**Constructor**: `constructor(address morpho)`

### 8.1 Deploy Script

Create `public-allocator/script/DeployPublicAllocator.s.sol`:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity 0.8.24;

import "forge-std/Script.sol";
import "../src/PublicAllocator.sol";

contract DeployPublicAllocator is Script {
    function run() external {
        uint256 deployerKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
        address morpho = vm.envAddress("MORPHO_ADDRESS");

        vm.startBroadcast(deployerKey);

        PublicAllocator pa = new PublicAllocator(morpho);
        console.log("PublicAllocator:", address(pa));

        vm.stopBroadcast();
    }
}
```

### 8.2 Deploy

```bash
cd ~/morpho/public-allocator
source ~/morpho/.env

forge script script/DeployPublicAllocator.s.sol:DeployPublicAllocator \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast --verify \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  -vvv
```

**After deploy**: Update `~/morpho/.env` → `PUBLIC_ALLOCATOR_ADDRESS`

### 8.3 Configure (after Vault V2 is deployed in Phase 10)

```bash
source ~/morpho/.env

# Set admin for the vault (so you can manage flow caps)
cast send "$PUBLIC_ALLOCATOR_ADDRESS" \
  "setAdmin(address,address)" "$VAULT_V2_ADDRESS" "$DEPLOYER_ADDRESS" \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY"

# Set flow caps for each market the vault allocates to
# FlowCaps control how much liquidity can flow in/out per reallocation
# Format: setFlowCaps(address vault, (bytes32 id, ((uint128 maxIn, uint128 maxOut))) [] calldata)
cast send "$PUBLIC_ALLOCATOR_ADDRESS" \
  "setFlowCaps(address,(bytes32,((uint128,uint128)))[])" \
  "$VAULT_V2_ADDRESS" \
  "[($MARKET_ID_WETH_USDC,((1000000000000,1000000000000)))]" \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY"
```

---

## Phase 9: Deploy Pre-Liquidation

Soft liquidation mechanism — protects borrowers before reaching the hard liquidation threshold.

**Source**: `pre-liquidation/src/PreLiquidationFactory.sol`
**Solidity**: `0.8.27`
**Factory constructor**: `constructor(address morpho)`

**`createPreLiquidation` signature**:
```solidity
function createPreLiquidation(
    Id id,                                    // Market ID (bytes32 wrapper type)
    PreLiquidationParams calldata params      // Configuration struct
) external returns (IPreLiquidation)
```

**`PreLiquidationParams` struct**:
```solidity
struct PreLiquidationParams {
    uint256 preLltv;                // LTV threshold to trigger pre-liquidation
    uint256 preLCF1;                // Close factor at preLltv
    uint256 preLCF2;                // Close factor at LLTV (hard liquidation threshold)
    uint256 preLIF1;                // Liquidation incentive at preLltv
    uint256 preLIF2;                // Liquidation incentive at LLTV
    address preLiquidationOracle;   // Oracle for price (usually same as market oracle)
}
```

### 9.1 Deploy Script

Create `pre-liquidation/script/DeployPreLiquidation.s.sol`:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity 0.8.27;

import "forge-std/Script.sol";
import "../src/PreLiquidationFactory.sol";

contract DeployPreLiquidation is Script {
    function run() external {
        uint256 deployerKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
        address morpho = vm.envAddress("MORPHO_ADDRESS");

        vm.startBroadcast(deployerKey);

        PreLiquidationFactory factory = new PreLiquidationFactory(morpho);
        console.log("PreLiquidationFactory:", address(factory));

        vm.stopBroadcast();
    }
}
```

### 9.2 Deploy

```bash
cd ~/morpho/pre-liquidation
source ~/morpho/.env

forge script script/DeployPreLiquidation.s.sol:DeployPreLiquidation \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast --verify \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  -vvv
```

**After deploy**: Update `~/morpho/.env` → `PRE_LIQUIDATION_FACTORY_ADDRESS`

### 9.3 Create Pre-Liquidation Instance Per Market

```bash
source ~/morpho/.env

# Example: WETH/USDC market with pre-liquidation at 83% LTV
# (market LLTV is 86%, so pre-liq triggers 3% earlier)
cast send "$PRE_LIQUIDATION_FACTORY_ADDRESS" \
  "createPreLiquidation(bytes32,(uint256,uint256,uint256,uint256,uint256,address))" \
  "$MARKET_ID_WETH_USDC" \
  "(830000000000000000,200000000000000000,500000000000000000,1010000000000000000,1050000000000000000,$ORACLE_WETH_USDC_ADDRESS)" \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY"

# Parameters explained:
#   preLltv:  0.83e18  → Triggers at 83% LTV (before 86% hard liquidation)
#   preLCF1:  0.20e18  → 20% close factor at preLltv
#   preLCF2:  0.50e18  → 50% close factor at LLTV
#   preLIF1:  1.01e18  → 1% liquidation incentive at preLltv
#   preLIF2:  1.05e18  → 5% liquidation incentive at LLTV
#   oracle:   Same oracle as the market
```

---

## Phase 10: Deploy Vault V2 (Lender Pooling)

Use the official `vault-v2-deployment` scripts for production-grade deployment with timelocks and security.

> **Note**: The `vault-v2-deployment` repo is marked as educational/demonstration and not audited. Review all scripts before mainnet use.

### 10.1 Configure Environment

```bash
cd ~/morpho/vault-v2-deployment
source ~/morpho/.env

# Create deployment-specific .env
cat > .env << ENVEOF
# ── From master .env ──
MORPHO_ADDRESS=$MORPHO_ADDRESS
IRM_ADDRESS=$IRM_ADDRESS
DEPLOYER_PRIVATE_KEY=$DEPLOYER_PRIVATE_KEY
BASE_RPC_URL=$BASE_RPC_URL
BASESCAN_API_KEY=$BASESCAN_API_KEY

# ── Role addresses ──
# (Can all be deployer initially, transfer to multisig later in Phase 15)
OWNER=$DEPLOYER_ADDRESS
CURATOR=$DEPLOYER_ADDRESS
ALLOCATOR=$DEPLOYER_ADDRESS
SENTINEL=0x0000000000000000000000000000000000000000

# ── Asset ──
ASSET=0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913

# ── Infrastructure (deploy factories first if needed) ──
VAULT_V2_FACTORY=$VAULT_V2_FACTORY_ADDRESS
MORPHO_MARKET_V1_ADAPTER_V2_FACTORY=$ADAPTER_FACTORY_ADDRESS
ADAPTER_REGISTRY=
ORACLE_FACTORY_ADDRESS=$ORACLE_FACTORY_ADDRESS

# ── First market (bytes32(0) to skip) ──
MARKET_ID=$MARKET_ID_WETH_USDC
COLLATERAL_TOKEN_CAP=1000000000000
MARKET_CAP=1000000000000

# ── Timelocks ──
# Minimum 3 days (259200s) for Morpho listing eligibility
TIMELOCK_DURATION=604800
ADAPTER_TIMELOCK_DURATION=259200

# ── Dead deposit (inflation attack protection) ──
DEAD_DEPOSIT_AMOUNT=1000000000
ENVEOF
```

### 10.2 Option A: Use Official Deploy Script

```bash
cd ~/morpho/vault-v2-deployment
source .env

forge script script/DeployVaultV2WithMarketAdapter.s.sol \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast --verify \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  -vvv
```

This runs the full deployment sequence:
1. Deploy VaultV2 via factory
2. Set temporary permissions
3. Deploy MorphoMarketV1AdapterV2
4. Submit timelocked config (allocator, adapter registry, add adapter)
5. Execute immediate config
6. Set final roles (owner, curator, allocator, sentinel)
7. Configure market & liquidity adapter
8. Dead deposit (1e9 wei to `0xdead` for inflation attack protection)
9. Vault timelocks (7 days)
10. Adapter timelocks (3 days)

### 10.3 Option B: Manual Deployment

If the official script needs customization, deploy components individually.

**VaultV2Factory**: Stateless, no constructor args.
**VaultV2**: Created via `factory.createVaultV2(address owner, address asset, bytes32 salt)`

Create `vault-v2/script/DeployVault.s.sol`:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity ^0.8.28;

import "forge-std/Script.sol";
import "../src/VaultV2.sol";
import "../src/VaultV2Factory.sol";
import "../src/adapters/MorphoMarketV1AdapterV2.sol";
import "../src/adapters/MorphoMarketV1AdapterV2Factory.sol";

contract DeployVault is Script {
    function run() external {
        uint256 deployerKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
        address owner = vm.addr(deployerKey);
        address morpho = vm.envAddress("MORPHO_ADDRESS");
        address irm = vm.envAddress("IRM_ADDRESS");
        address USDC = 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913;

        vm.startBroadcast(deployerKey);

        // Step 1: Deploy factories
        VaultV2Factory vaultFactory = new VaultV2Factory();
        MorphoMarketV1AdapterV2Factory adapterFactory =
            new MorphoMarketV1AdapterV2Factory(morpho, irm);

        // Step 2: Deploy vault via factory
        VaultV2 vault = vaultFactory.createVaultV2(owner, USDC, bytes32(uint256(1)));

        // Step 3: Deploy adapter for this vault
        MorphoMarketV1AdapterV2 adapter =
            adapterFactory.createMorphoMarketV1AdapterV2(address(vault));

        // Step 4: Configure vault roles and adapter
        vault.setCurator(owner);
        vault.setIsAllocator(owner, true);
        vault.addAdapter(address(adapter));

        vm.stopBroadcast();

        console.log("VaultV2Factory:", address(vaultFactory));
        console.log("AdapterFactory:", address(adapterFactory));
        console.log("VaultV2:", address(vault));
        console.log("Adapter:", address(adapter));
    }
}
```

```bash
cd ~/morpho/vault-v2
source ~/morpho/.env

forge script script/DeployVault.s.sol:DeployVault \
  --rpc-url "$BASE_RPC_URL" \
  --private-key "$DEPLOYER_PRIVATE_KEY" \
  --broadcast --verify \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  -vvv
```

**After deploy**: Update `~/morpho/.env` → `VAULT_V2_FACTORY_ADDRESS`, `ADAPTER_FACTORY_ADDRESS`, `VAULT_V2_ADDRESS`, `ADAPTER_ADDRESS`

---

## Phase 11: Deploy eRPC (Fault-Tolerant RPC Proxy)

Caches and load-balances RPC requests. Reduces cost and improves reliability for the subgraph and frontend.

**Source**: `erpc/erpc` on GitHub (Go project)
**Binary**: `bin/erpc-server` after `make build`
**Config**: `erpc.yaml` (YAML) or `erpc.ts` (TypeScript)
**Requires**: Go 1.25+ (per `go.mod`)

### 11.1 Configure

```bash
cd ~/morpho/erpc
cp erpc.dist.yaml erpc.yaml
```

Edit `erpc.yaml` — the config uses `httpHostV4`/`httpPortV4` (not `httpHost`/`httpPort`):

```yaml
logLevel: warn

server:
  httpHostV4: "0.0.0.0"
  httpPortV4: 4000
  maxTimeout: "50s"

metrics:
  hostV4: "0.0.0.0"
  portV4: 4001

projects:
  - id: morpho-base
    networks:
      - architecture: evm
        evm:
          chainId: 8453
    upstreams:
      - id: base-public
        endpoint: https://mainnet.base.org
        type: evm
        rateLimitBudget: base-public-budget
      # Add paid RPCs for production reliability:
      # - id: alchemy-base
      #   endpoint: alchemy://YOUR_ALCHEMY_API_KEY
      #   type: evm
      # - id: infura-base
      #   endpoint: infura://YOUR_INFURA_API_KEY
      #   type: evm

rateLimiters:
  budgets:
    - id: base-public-budget
      rules:
        - method: "*"
          maxCount: 50
          period: 1s

database:
  evmJsonRpcCache:
    connectors:
      - id: memory-cache
        driver: memory
        memory:
          maxItems: 100000
    policies:
      - network: "evm:8453"
        finality: finalized
        connector: memory-cache
```

### 11.2 Build and Run as systemd Service

```bash
# Build the binary
cd ~/morpho/erpc
make build
# Produces: ./bin/erpc-server

# Verify it runs
./bin/erpc-server &
sleep 2
curl -s http://localhost:4000 -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'
# Should return: {"jsonrpc":"2.0","id":1,"result":"0x2105"}
kill %1
```

Create the systemd service (note: replace `YOUR_USER` with the actual username):

```bash
# Get your username for the service file
CURRENT_USER=$(whoami)
CURRENT_HOME=$(eval echo ~$CURRENT_USER)

sudo tee /etc/systemd/system/erpc.service << EOF
[Unit]
Description=eRPC Fault-Tolerant RPC Proxy
After=network.target

[Service]
Type=simple
User=${CURRENT_USER}
WorkingDirectory=${CURRENT_HOME}/morpho/erpc
ExecStart=${CURRENT_HOME}/morpho/erpc/bin/erpc-server
Restart=always
RestartSec=5
Environment=HOME=${CURRENT_HOME}

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable erpc
sudo systemctl start erpc
```

### 11.3 Verify

```bash
# Check service status
sudo systemctl status erpc

# Test RPC — should return Base chain ID (0x2105 = 8453)
curl -s http://localhost:4000 \
  -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'

# Check metrics endpoint
curl -s http://localhost:4001/metrics | head -5
```

From now on, use `http://localhost:4000` as your RPC URL for the subgraph and frontend.

---

## Phase 12: Deploy Subgraph (Event Indexer)

Indexes all Morpho events for the frontend to query via GraphQL.

**Source**: `morpho-org/morpho-blue-subgraph`
**Stack**: TheGraph (graph-node + PostgreSQL + IPFS via Docker Compose)
**Package manager**: yarn (classic)

### 12.1 Docker Compose Services

The repo's `docker-compose.yml` includes:
- **graph-node** (`graphprotocol/graph-node`) — ports 8000 (queries), 8001 (WS), 8020 (admin), 8030 (index status)
- **ipfs** (`ipfs/kubo:latest`) — ports 4001, 5001
- **postgres** (`postgres`) — internal only

### 12.2 Configure for Base

```bash
cd ~/morpho/morpho-blue-subgraph
```

Edit `networks.json` — add/update the `base` section with your deployed addresses:

```json
{
  "base": {
    "MorphoBlue": {
      "address": "YOUR_MORPHO_ADDRESS",
      "startBlock": YOUR_DEPLOY_BLOCK_NUMBER
    }
  }
}
```

> **How to find your deploy block**: Check the Morpho deployment transaction on BaseScan, or run:
> ```bash
> cast receipt YOUR_DEPLOY_TX_HASH --rpc-url "$BASE_RPC_URL" | grep blockNumber
> ```

Edit `subgraph.yaml` — set `network: base` and update contract addresses and start blocks to match `networks.json`.

### 12.3 Set RPC URL for Docker

The graph-node needs an RPC URL. Set it via environment variable:

```bash
# Export the RPC URL that docker-compose will use
# Use eRPC on the host machine (host.docker.internal resolves to host from Docker)
export MAINNET_RPC_URL="base:http://host.docker.internal:4000"
```

If the `docker-compose.yml` hardcodes the ethereum env var, create an override:

```bash
cat > docker-compose.override.yml << 'EOF'
services:
  graph-node:
    environment:
      ethereum: "base:http://host.docker.internal:4000"
    extra_hosts:
      - "host.docker.internal:host-gateway"
EOF
```

> **Note on `host.docker.internal`**: On Linux, you may need the `extra_hosts` entry above. On Docker Desktop (Mac/Windows), it works automatically.

### 12.4 Start and Deploy

```bash
cd ~/morpho/morpho-blue-subgraph

# Start infrastructure
docker compose up -d

# Wait for graph-node to initialize (check logs)
docker compose logs -f graph-node &
sleep 30

# Install dependencies and generate types
yarn install
yarn codegen

# Create the subgraph namespace on local graph-node
yarn create-local
# Equivalent to: graph create --node http://localhost:8020/ morpho-org/morpho-blue

# Deploy the subgraph
yarn deploy-local
# Equivalent to: graph deploy --node http://localhost:8020/ --ipfs http://localhost:5001 morpho-org/morpho-blue
```

### 12.5 Verify

```bash
# Query the subgraph (may take minutes to index depending on start block)
curl -s http://localhost:8000/subgraphs/name/morpho-org/morpho-blue \
  -H "Content-Type: application/json" \
  -d '{"query": "{ markets(first: 5) { id } }"}'

# Check indexing status
curl -s http://localhost:8030/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ indexingStatuses { synced health chains { latestBlock { number } } } }"}'
```

### 12.6 Ensure Auto-Restart

```bash
# Set all subgraph containers to restart automatically
docker update --restart=always $(docker compose ps -q)
```

---

## Phase 13: Build SDKs

Build the TypeScript SDKs for use by the frontend.

**Source**: `morpho-org/sdks` — pnpm monorepo
**Packages include**:
- `@morpho-org/blue-sdk` — Protocol entities and types
- `@morpho-org/blue-sdk-viem` — Viem-based on-chain data fetching
- `@morpho-org/blue-sdk-wagmi` — Wagmi hooks for React
- `@morpho-org/bundler-sdk-viem` — Bundler3 transaction building
- `@morpho-org/simulation-sdk` — Pre-transaction simulation
- `@morpho-org/simulation-sdk-wagmi` — Simulation with Wagmi
- `@morpho-org/liquidity-sdk-viem` — PublicAllocator integration
- `@morpho-org/liquidation-sdk-viem` — Liquidation helpers
- `@morpho-org/migration-sdk-viem` — Migration utilities

```bash
cd ~/morpho/sdks

# Install dependencies
pnpm install

# Configure RPC (used for tests/code generation)
echo "BASE_RPC_URL=http://localhost:4000" > .env

# Build all packages
pnpm build
```

> **Note**: The SDKs may reference specific deployed Morpho addresses for known chains. If deploying to a chain/address not already configured in the SDK, you may need to update chain config files within the SDK packages. Check `packages/blue-sdk/src/chain/` for chain-specific configurations.

---

## Phase 14: Deploy Frontend (earn-basic-app)

**Source**: `morpho-org/earn-basic-app` — React + Vite + TypeScript
**Dependencies**: `@morpho-org/blue-sdk-*`, `@rainbow-me/rainbowkit`, `wagmi`, `viem`, `@apollo/client`, TailwindCSS v4

### 14.1 Configure

```bash
cd ~/morpho/earn-basic-app
pnpm install
```

Update the app configuration to point to your deployed contracts and infrastructure:

- **RPC endpoint**: `http://YOUR_VPS_IP:4000` (eRPC) — or use the Nginx proxy path `/rpc`
- **Subgraph endpoint**: `http://YOUR_VPS_IP:8000/subgraphs/name/morpho-org/morpho-blue` — or Nginx `/subgraph`
- **Contract addresses**: Your deployed Morpho, Vault, Bundler addresses
- **Default vault**: Your `VAULT_V2_ADDRESS`
- **Chain ID**: `8453` (Base)

> **Where to configure**: Check `.env.example` or `src/config/` for environment variables. The app likely uses `VITE_*` prefixed env vars for Vite.

### 14.2 Build

```bash
pnpm build
# Output in dist/ — static files ready for deployment
```

### 14.3 Serve with Nginx

```bash
sudo apt install -y nginx

# Get username for paths
CURRENT_USER=$(whoami)
CURRENT_HOME=$(eval echo ~$CURRENT_USER)

# Create Nginx config
sudo tee /etc/nginx/sites-available/morpho << NGINXEOF
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;

    # Frontend (static SPA)
    location / {
        root ${CURRENT_HOME}/morpho/earn-basic-app/dist;
        try_files \$uri \$uri/ /index.html;
    }

    # eRPC proxy (so frontend doesn't need direct port access)
    location /rpc {
        proxy_pass http://127.0.0.1:4000;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }

    # Subgraph proxy
    location /subgraph/ {
        proxy_pass http://127.0.0.1:8000/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
NGINXEOF

sudo ln -sf /etc/nginx/sites-available/morpho /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl restart nginx
```

---

## Phase 15: Production Hardening

### 15.1 Role Separation

```
┌──────────┐  ┌──────────┐  ┌───────────┐  ┌───────────┐
│  Owner   │  │ Curator  │  │ Allocator │  │ Sentinel  │
│(Multisig)│  │(Multisig)│  │  (Bot)    │  │(Emergency)│
└────┬─────┘  └────┬─────┘  └─────┬─────┘  └─────┬─────┘
     │             │               │               │
     │ Morpho:     │ Vault:        │ Vault:         │ Vault:
     │ setOwner    │ addAdapter    │ allocate       │ emergency
     │ enableIrm   │ setCaps       │ setMaxRate     │   withdraw
     │ enableLltv  │               │ reallocate     │
     │             │               │                │
     │ Vault:      │               │                │
     │ setCurator  │               │                │
     │ setAllocator│               │                │
     ▼             ▼               ▼                ▼
  ┌─────────────────────────────────────────────────────┐
  │                   PROTOCOL                           │
  └─────────────────────────────────────────────────────┘
```

### 15.2 Transfer Ownership to Multisig

```bash
source ~/morpho/.env

# ── Step 1: Deploy a Gnosis Safe on Base first ──
# Go to https://app.safe.global and create a Safe on Base (chain 8453)
# Set your SAFE_ADDRESS:
SAFE_ADDRESS=0x_YOUR_GNOSIS_SAFE_ADDRESS

# ── Step 2: Transfer Morpho Blue ownership ──
cast send "$MORPHO_ADDRESS" "setOwner(address)" "$SAFE_ADDRESS" \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY"

# Verify
cast call "$MORPHO_ADDRESS" "owner()(address)" --rpc-url "$BASE_RPC_URL"
# Should return SAFE_ADDRESS

# ── Step 3: Transfer Vault V2 ownership ──
# Check the exact function name in the vault contract (may be transferOwnership or setOwner)
cast send "$VAULT_V2_ADDRESS" "transferOwnership(address)" "$SAFE_ADDRESS" \
  --rpc-url "$BASE_RPC_URL" --private-key "$DEPLOYER_PRIVATE_KEY"
```

### 15.3 Key Management

```bash
# Import deployer key to encrypted Foundry keystore
cast wallet import deployer --interactive
# Enter private key when prompted, then set an encryption password

# Verify it was imported
cast wallet list

# Delete the plaintext .env private key
# (edit ~/morpho/.env and remove DEPLOYER_PRIVATE_KEY line)

# Future deploys use the encrypted keystore:
forge script ... --account deployer
# (prompts for password each time)
```

### 15.4 Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp    # Nginx HTTP
sudo ufw allow 443/tcp   # Nginx HTTPS (after SSL setup)
# Do NOT expose 4000 (eRPC) or 8000 (subgraph) directly — access via Nginx proxy
sudo ufw enable
sudo ufw status
```

### 15.5 SSL (Let's Encrypt)

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d YOUR_DOMAIN

# Certbot auto-renews via systemd timer. Verify:
sudo systemctl status certbot.timer
```

### 15.6 Monitoring

```bash
cat > ~/morpho/monitor.sh << 'SCRIPT'
#!/bin/bash
source ~/morpho/.env
LOG=/var/log/morpho-monitor.log
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "=== $TIMESTAMP ===" >> "$LOG"

# ── On-chain checks via eRPC ──
RPC="http://localhost:4000"

OWNER=$(cast call "$MORPHO_ADDRESS" "owner()(address)" --rpc-url "$RPC" 2>&1)
echo "Morpho owner: $OWNER" >> "$LOG"

TOTAL=$(cast call "$VAULT_V2_ADDRESS" "totalAssets()(uint256)" --rpc-url "$RPC" 2>&1)
echo "Vault USDC total: $TOTAL" >> "$LOG"

PRICE=$(cast call "$ORACLE_WETH_USDC_ADDRESS" "price()(uint256)" --rpc-url "$RPC" 2>&1)
echo "WETH/USDC oracle: $PRICE" >> "$LOG"

# ── Service health checks ──
ERPC_STATUS=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$RPC" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}')
echo "eRPC HTTP status: $ERPC_STATUS" >> "$LOG"

SUBGRAPH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  "http://localhost:8000/subgraphs/name/morpho-org/morpho-blue" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ _meta { block { number } } }"}')
echo "Subgraph HTTP status: $SUBGRAPH_STATUS" >> "$LOG"

# ── Alert on failure ──
if [ "$ERPC_STATUS" != "200" ] || [ "$SUBGRAPH_STATUS" != "200" ]; then
  echo "ALERT: Service degraded! eRPC=$ERPC_STATUS Subgraph=$SUBGRAPH_STATUS" >> "$LOG"
  # Optional: send alert via webhook, email, etc.
fi
SCRIPT

chmod +x ~/morpho/monitor.sh

# Create log file with proper permissions
sudo touch /var/log/morpho-monitor.log
sudo chown $(whoami) /var/log/morpho-monitor.log

# Add to crontab (runs every 5 minutes)
(crontab -l 2>/dev/null; echo "*/5 * * * * $HOME/morpho/monitor.sh") | crontab -

# Verify cron entry
crontab -l
```

---

## Deployment Order (Complete)

```
 SMART CONTRACTS (Base Chain)
 ════════════════════════════
 Step  1 → Morpho Blue              constructor(owner)
 Step  2 → AdaptiveCurveIrm         constructor(morpho)
 Step  3 → Oracle Factory           no constructor args
 Step  4 → Oracles (per pair)       factory.create(feeds, decimals, salt)
 Step  5 → Enable IRM               morpho.enableIrm(irm)
 Step  6 → Enable LLTVs             morpho.enableLltv(x) × N
 Step  7 → Create Markets           morpho.createMarket(params) × N
 Step  8 → Set Fees                 morpho.setFee(params, fee) × N
 Step  9 → Bundler3                 no constructor args
 Step 10 → GeneralAdapter1          constructor(bundler, morpho, weth)
 Step 11 → PublicAllocator          constructor(morpho)
 Step 12 → PreLiquidationFactory    constructor(morpho)
 Step 13 → PreLiquidation instances factory.createPreLiquidation(marketId, params)
 Step 14 → VaultV2 Factory          no constructor args
 Step 15 → AdapterV2 Factory        constructor(morpho, irm)
 Step 16 → VaultV2 instance         factory.createVaultV2(owner, usdc, salt)
 Step 17 → MarketAdapter instance   factory.createMorphoMarketV1AdapterV2(vault)
 Step 18 → Configure Vault          setCurator, setIsAllocator, addAdapter
 Step 19 → Configure PublicAlloc    setAdmin, setFlowCaps
 Step 20 → Dead deposit             vault.deposit(1e9, 0xdead)
 Step 21 → Set timelocks            vault timelock: 7d, adapter timelock: 3d
 Step 22 → Verify all on Basescan   forge verify-contract for each
 Step 23 → Transfer ownership       → Gnosis Safe multisig

 VPS INFRASTRUCTURE
 ════════════════════════════
 Step 24 → eRPC                     erpc.yaml → make build → systemd service
 Step 25 → Subgraph                 docker compose (graph-node + postgres + ipfs)
 Step 26 → SDKs                     pnpm install && pnpm build
 Step 27 → Frontend                 pnpm install && pnpm build → Nginx
 Step 28 → SSL                      certbot --nginx
 Step 29 → Firewall                 ufw (allow 22, 80, 443 only)
 Step 30 → Monitoring               cron job every 5 min
```

---

## Estimated Costs

### Smart Contracts (Base L2)

| Component | Approx Gas | Cost (at ~$0.01/M gas) |
|-----------|-----------|------|
| Morpho Blue | ~4M | < $0.05 |
| AdaptiveCurveIrm | ~1M | < $0.02 |
| Oracle Factory + 2 oracles | ~1.5M | < $0.03 |
| Market setup (2 markets) | ~600K | < $0.01 |
| Bundler3 + GeneralAdapter1 | ~3M | < $0.05 |
| PublicAllocator | ~1M | < $0.02 |
| PreLiquidationFactory + 2 instances | ~2M | < $0.03 |
| VaultV2 Factory + Vault + Adapter | ~4M | < $0.06 |
| Config transactions | ~1M | < $0.02 |
| **Total on-chain** | **~18M** | **< $1.00** |

> **Note**: Base L2 gas costs fluctuate. Check current gas prices at [basescan.org/gastracker](https://basescan.org/gastracker).

### VPS Monthly

| Service | Cost |
|---------|------|
| VPS (16GB RAM, 100GB SSD) | ~$40–80/mo |
| Paid RPC (Alchemy Growth / Infura) | ~$0–50/mo |
| Domain + SSL (Let's Encrypt = free) | ~$10/yr |
| **Total infra** | **~$50–130/mo** |

---

## Full Checklist

### Smart Contracts
- [ ] Morpho Blue deployed + verified on BaseScan
- [ ] AdaptiveCurveIrm deployed + verified
- [ ] Oracle Factory deployed
- [ ] WETH/USDC Oracle deployed + `price()` returns sane value
- [ ] cbETH/USDC Oracle deployed + `price()` returns sane value
- [ ] IRM enabled on Morpho (`isIrmEnabled(irm)` returns true)
- [ ] LLTVs enabled (80%, 86%, 91.5%) (`isLltvEnabled(lltv)` returns true)
- [ ] Markets created with 10% fees
- [ ] Fee recipient set (`feeRecipient()` returns correct address)
- [ ] Market IDs computed and saved
- [ ] Bundler3 deployed + verified
- [ ] GeneralAdapter1 deployed + verified
- [ ] PublicAllocator deployed + verified
- [ ] PreLiquidationFactory deployed + verified
- [ ] PreLiquidation instances created per market
- [ ] VaultV2 Factory deployed
- [ ] USDC Vault deployed + verified
- [ ] MorphoMarketV1AdapterV2 deployed
- [ ] Vault configured (curator, allocator, adapter)
- [ ] PublicAllocator configured (admin, flow caps)
- [ ] Dead deposit made (1e9 wei to 0xdead — inflation protection)
- [ ] Timelocks set (vault: 7d, adapter: 3d)

### Infrastructure
- [ ] VPS provisioned (16GB RAM, 100GB SSD, Ubuntu 22.04+)
- [ ] Foundry installed (`forge --version` works)
- [ ] Node.js 22 LTS installed
- [ ] Go 1.25+ installed
- [ ] Docker + Docker Compose installed
- [ ] pnpm + yarn installed
- [ ] Firewall configured (only 22, 80, 443 open)
- [ ] eRPC running as systemd service (port 4000)
- [ ] eRPC returns `eth_chainId` = `0x2105`
- [ ] Subgraph Docker containers running (graph-node, ipfs, postgres)
- [ ] Subgraph indexing and responding to GraphQL queries
- [ ] SDKs built successfully
- [ ] Frontend built and served via Nginx
- [ ] SSL enabled (Let's Encrypt / certbot)
- [ ] Monitoring cron active (every 5 min)

### Security
- [ ] All contracts verified on BaseScan
- [ ] Morpho ownership transferred → Gnosis Safe multisig
- [ ] Vault ownership transferred → Gnosis Safe multisig
- [ ] Private keys removed from plaintext `.env` files
- [ ] Deployer key in encrypted Foundry keystore only
- [ ] No ports exposed except 22/80/443
- [ ] eRPC (4000) and subgraph (8000) only accessible via Nginx proxy
- [ ] Log rotation configured for `/var/log/morpho-monitor.log`

---

## User Flows (Post-Deployment)

### Lender (via Frontend or Bundler3)
```
1. Connect wallet on earn-basic-app (RainbowKit)
2. Approve USDC → Vault V2
3. Deposit USDC → receive vault shares (ERC4626)
4. Vault curator allocates USDC → Morpho Blue markets
5. Earn interest from borrowers (auto-compounding)
6. Withdraw anytime (USDC + accrued interest)
```

### Borrower (via Morpho Blue directly)
```
1. Supply collateral (WETH/cbETH) → Morpho Blue market
2. Borrow USDC against collateral (up to LLTV)
3. Repay USDC + interest
4. Withdraw collateral
```

### Liquidator
```
1. Monitor positions via subgraph GraphQL queries
2. Pre-liquidate unhealthy positions (soft, via PreLiquidation contract)
   - Triggers at preLltv (e.g. 83%), before hard liquidation (86%)
   - Smaller penalty for borrower (1-5% incentive)
3. Hard liquidate via Morpho Blue if position reaches LLTV
4. Earn liquidation incentive
```

### Public Reallocator
```
1. Anyone can call PublicAllocator.reallocateTo()
2. Moves idle vault liquidity to higher-utilization markets
3. Improves capital efficiency for all lenders
4. Subject to flow caps set by vault admin
```

---

## Troubleshooting

### Forge script fails with "insufficient funds"
```bash
# Check deployer ETH balance on Base
cast balance "$DEPLOYER_ADDRESS" --rpc-url "$BASE_RPC_URL" --ether
# Bridge more ETH from L1 if needed
```

### Subgraph not indexing
```bash
# Check graph-node logs
docker compose logs graph-node | tail -50

# Common issues:
# 1. RPC URL unreachable from Docker → use host.docker.internal
# 2. Wrong startBlock → set to your actual deployment block
# 3. Network mismatch → ensure subgraph.yaml says "base"
```

### eRPC returning errors
```bash
# Check systemd logs
sudo journalctl -u erpc -f

# Test direct RPC bypass
curl -s https://mainnet.base.org -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'
```

### Contract verification fails
```bash
# Retry with explicit compiler settings matching foundry.toml
forge verify-contract "$CONTRACT_ADDRESS" src/Contract.sol:ContractName \
  --chain base \
  --etherscan-api-key "$BASESCAN_API_KEY" \
  --compiler-version "v0.8.19+commit.7dd6d404" \
  --num-of-optimizations 999999
```