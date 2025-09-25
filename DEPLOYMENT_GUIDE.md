# IU2U Protocol - Deployment Guide

## Overview

This guide provides step-by-step instructions for deploying the IU2U Protocol across multiple blockchain networks. The deployment process involves setting up core contracts, configuring relayer infrastructure, and establishing cross-chain communication.

## Prerequisites

### System Requirements
- **Node.js**: >= 18.0.0
- **npm**: >= 8.0.0
- **Git**: Latest version
- **Operating System**: Windows/macOS/Linux

### Required Tools
- **Hardhat**: Development framework
- **Ethers.js**: Blockchain interaction library
- **OpenZeppelin**: Security-audited contract libraries

### Funding Requirements

| Chain | Estimated Cost | Purpose |
|-------|----------------|---------|
| U2U | 0.1 U2U | Core contracts + vault deployment |
| Ethereum | 0.05 ETH | Gateway + meta-tx deployment |
| BSC | 0.01 BNB | Gateway deployment |
| Polygon | 0.3 MATIC | Gateway deployment |
| Avalanche | 20 AVAX | Gateway deployment |
| Arbitrum | 0.01 ETH | Gateway deployment |
| Optimism | 0.01 ETH | Gateway deployment |
| Base | 0.01 ETH | Gateway deployment |

## Environment Setup

### 1. Clone Repository

```bash
# Clone the IU2U Contracts repository
git clone https://github.com/DINetworks/IU2U-Contracts.git
cd IU2U-Contracts

# Install dependencies
npm install

# Install relayer dependencies
cd relayer
npm install
cd ..
```

### 2. Environment Configuration

Create `.env` file in the root directory:

```bash
# Private keys (deployer and relayer)
PRIVATE_KEY=your_deployer_private_key_here
RELAYER_PRIVATE_KEY=your_relayer_private_key_here

# RPC Endpoints
U2U_RPC=https://rpc.testnet.ms
ETHEREUM_RPC=https://mainnet.infura.io/v3/YOUR_INFURA_KEY
BSC_RPC=https://bsc-dataseed1.binance.org
POLYGON_RPC=https://polygon-rpc.com
AVALANCHE_RPC=https://api.avax.network/ext/bc/C/rpc
ARBITRUM_RPC=https://arb1.arbitrum.io/rpc
OPTIMISM_RPC=https://mainnet.optimism.io
BASE_RPC=https://mainnet.base.org

# API Keys for contract verification
ETHERSCAN_API_KEY=your_etherscan_api_key
BSCSCAN_API_KEY=your_bscscan_api_key
POLYGONSCAN_API_KEY=your_polygonscan_api_key
SNOWTRACE_API_KEY=your_snowtrace_api_key
ARBISCAN_API_KEY=your_arbiscan_api_key
OPTIMISTIC_API_KEY=your_optimistic_etherscan_api_key
BASESCAN_API_KEY=your_basescan_api_key

# Relayer configuration
PORT=3001
CORS_ORIGINS=http://localhost:3000,https://your-dapp.com
```

### 3. Network Configuration

Verify `hardhat.config.js` contains all target networks:

```javascript
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

module.exports = {
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200
      }
    }
  },
  networks: {
    // U2U Networks
    u2u: {
      chainId: 2484,
      url: process.env.CROSSFI_RPC,
      accounts: [process.env.PRIVATE_KEY]
    },
    u2uMainnet: {
      chainId: 39,
      url: "https://rpc.mainnet.ms",
      accounts: [process.env.PRIVATE_KEY]
    },
    
    // Ethereum Networks
    ethereum: {
      chainId: 1,
      url: process.env.ETHEREUM_RPC,
      accounts: [process.env.PRIVATE_KEY]
    },
    sepolia: {
      chainId: 11155111,
      url: "https://sepolia.infura.io/v3/" + process.env.INFURA_KEY,
      accounts: [process.env.PRIVATE_KEY]
    },
    
    // Other Networks
    bsc: {
      chainId: 56,
      url: process.env.BSC_RPC,
      accounts: [process.env.PRIVATE_KEY]
    },
    polygon: {
      chainId: 137,
      url: process.env.POLYGON_RPC,
      accounts: [process.env.PRIVATE_KEY]
    },
    avalanche: {
      chainId: 43114,
      url: process.env.AVALANCHE_RPC,
      accounts: [process.env.PRIVATE_KEY]
    },
    arbitrum: {
      chainId: 42161,
      url: process.env.ARBITRUM_RPC,
      accounts: [process.env.PRIVATE_KEY]
    },
    optimism: {
      chainId: 10,
      url: process.env.OPTIMISM_RPC,
      accounts: [process.env.PRIVATE_KEY]
    },
    base: {
      chainId: 8453,
      url: process.env.BASE_RPC,
      accounts: [process.env.PRIVATE_KEY]
    }
  },
  etherscan: {
    apiKey: {
      mainnet: process.env.ETHERSCAN_API_KEY,
      sepolia: process.env.ETHERSCAN_API_KEY,
      bsc: process.env.BSCSCAN_API_KEY,
      polygon: process.env.POLYGONSCAN_API_KEY,
      avalanche: process.env.SNOWTRACE_API_KEY,
      arbitrumOne: process.env.ARBISCAN_API_KEY,
      optimisticEthereum: process.env.OPTIMISTIC_API_KEY,
      base: process.env.BASESCAN_API_KEY
    }
  }
};
```

## Contract Deployment

### 1. Deploy Core Gateway System

#### Step 1: Deploy on U2U (Native Chain)

```bash
# Deploy IU2U Gateway and MetaTx Vault on U2U
npx hardhat run scripts/deploy-gmp.js --network u2u
npx hardhat run scripts/deploy-meta-tx.js --network u2u
```

Expected output:
```
Deploying IU2U Gateway on U2U...
IU2U Gateway deployed to: 0xAbcdef1234567890123456789012345678901234
MetaTxGasCreditVault deployed to: 0x1234567890123456789012345678901234567890
✅ U2U deployment complete
```

#### Step 2: Deploy on Destination Chains

```bash
# Deploy on Ethereum
npx hardhat run scripts/deploy-gmp.js --network ethereum
npx hardhat run scripts/deploy-meta-tx.js --network ethereum

# Deploy on BSC
npx hardhat run scripts/deploy-gmp.js --network bsc
npx hardhat run scripts/deploy-meta-tx.js --network bsc

# Deploy on Polygon
npx hardhat run scripts/deploy-gmp.js --network polygon
npx hardhat run scripts/deploy-meta-tx.js --network polygon

# Deploy on other chains as needed
npx hardhat run scripts/deploy-gmp.js --network avalanche
npx hardhat run scripts/deploy-gmp.js --network arbitrum
npx hardhat run scripts/deploy-gmp.js --network optimism
npx hardhat run scripts/deploy-gmp.js --network base
```

### 2. Deploy DEX Aggregation System

```bash
# Deploy CrossChainAggregator on each supported chain
npx hardhat run scripts/deploy-aggregator.js --network ethereum
npx hardhat run scripts/deploy-aggregator.js --network bsc
npx hardhat run scripts/deploy-aggregator.js --network polygon
# ... repeat for all chains
```

### 3. Contract Verification

```bash
# Verify contracts on block explorers
npx hardhat run scripts/verify-contracts.js --network ethereum
npx hardhat run scripts/verify-contracts.js --network bsc
npx hardhat run scripts/verify-contracts.js --network polygon
# ... repeat for all chains
```

### 4. Record Deployment Addresses

Create `deployment-addresses.json`:

```json
{
  "networks": {
    "u2u": {
      "chainId": 2484,
      "iu2uGateway": "0x...",
      "metaTxVault": "0x...",
      "metaTxGateway": "0x...",
      "aggregator": "0x..."
    },
    "ethereum": {
      "chainId": 1,
      "iu2uGateway": "0x...",
      "metaTxGateway": "0x...",
      "aggregator": "0x..."
    },
    "bsc": {
      "chainId": 56,
      "iu2uGateway": "0x...",
      "metaTxGateway": "0x...",
      "aggregator": "0x..."
    }
    // ... other networks
  },
  "relayers": {
    "gmpRelayer": "0x...",
    "metaTxRelayer": "0x..."
  },
  "oracles": {
    "u2u": "0x...",
    "ethereum": "0x...",
    "bsc": "0x..."
  }
}
```

## Relayer Setup

### 1. Configure GMP Relayer

Create `relayer/config.json`:

```json
{
  "relayerPrivateKey": "0x...",
  "chains": {
    "u2u": {
      "rpc": "https://rpc.testnet.ms",
      "iu2uAddress": "0x...",
      "startBlock": 1000000,
      "confirmations": 3
    },
    "ethereum": {
      "rpc": "https://mainnet.infura.io/v3/YOUR_KEY",
      "iu2uAddress": "0x...",
      "startBlock": 18500000,
      "confirmations": 12
    },
    "bsc": {
      "rpc": "https://bsc-dataseed1.binance.org",
      "iu2uAddress": "0x...",
      "startBlock": 32000000,
      "confirmations": 15
    },
    "polygon": {
      "rpc": "https://polygon-rpc.com",
      "iu2uAddress": "0x...",
      "startBlock": 48000000,
      "confirmations": 50
    }
  },
  "monitoring": {
    "enabled": true,
    "pollInterval": 5000,
    "maxRetries": 3,
    "alertThreshold": "1000000000000000000"
  }
}
```

### 2. Configure Meta-Transaction Relayer

Create `relayer/meta-tx-config.json`:

```json
{
  "relayerPrivateKey": "0x...",
  "chains": {
    "u2u": {
      "rpc": "https://rpc.testnet.ms",
      "vaultAddress": "0x...",
      "gatewayAddress": "0x...",
      "gasPrice": "auto"
    },
    "ethereum": {
      "rpc": "https://mainnet.infura.io/v3/YOUR_KEY",
      "gatewayAddress": "0x...",
      "gasPrice": "auto",
      "maxGasPrice": "100000000000"
    },
    "bsc": {
      "rpc": "https://bsc-dataseed1.binance.org",
      "gatewayAddress": "0x...",
      "gasPrice": "5000000000"
    }
  },
  "server": {
    "port": 3001,
    "corsOrigins": [
      "http://localhost:3000",
      "https://your-dapp.com"
    ]
  },
  "creditManagement": {
    "checkInterval": 30000,
    "minCreditBuffer": 100,
    "maxRetries": 3,
    "alertEmail": "admin@yourproject.com"
  }
}
```

### 3. Start Relayer Services

```bash
# Terminal 1: Start GMP Relayer
cd relayer
node IU2URelayer.js

# Terminal 2: Start Meta-Transaction Relayer
node MetaTxRelayer.js
```

## Post-Deployment Configuration

### 1. Configure Cross-Chain Connections

```bash
# Authorize relayers on all chains
npx hardhat run scripts/authorize-relayers.js --network u2u
npx hardhat run scripts/authorize-relayers.js --network ethereum
npx hardhat run scripts/authorize-relayers.js --network bsc
# ... repeat for all chains
```

### 2. Configure Meta-Transaction Vault

```bash
# Authorize gateways to consume credits
npx hardhat run scripts/configure-vault.js --network u2u
```

Script content example:
```javascript
async function main() {
  const vaultAddress = "0x..."; // Your vault address
  const gatewayAddresses = [
    "0x...", // Ethereum gateway
    "0x...", // BSC gateway
    "0x..."  // Polygon gateway
  ];
  
  const vault = await ethers.getContractAt("MetaTxGasCreditVault", vaultAddress);
  
  for (const gateway of gatewayAddresses) {
    await vault.setGatewayAuthorization(gateway, true);
    console.log(`Authorized gateway: ${gateway}`);
  }
}
```

### 3. Configure Chain Registry

```bash
# Update chain configurations
npx hardhat run scripts/update-chains.js --network u2u
```

### 4. Initial Token Supply & Testing

```bash
# Mint initial IU2U tokens for testing
npx hardhat run scripts/mint-initial-supply.js --network u2u

# Test cross-chain functionality
npx hardhat run scripts/test-cross-chain.js --network u2u
```

## Testing & Validation

### 1. Contract Deployment Tests

```bash
# Run deployment validation tests
npm run test:deployment

# Check all contracts are deployed correctly
npx hardhat run scripts/validate-deployment.js --network ethereum
```

### 2. Cross-Chain Functionality Tests

```bash
# Test cross-chain message passing
npx hardhat run scripts/test-gmp.js --network u2u

# Test meta-transaction execution
npx hardhat run scripts/test-meta-tx.js --network ethereum

# Test DEX aggregation
npx hardhat run scripts/test-aggregator.js --network polygon
```

### 3. Relayer Health Checks

```bash
# Check relayer status
curl http://localhost:3001/health

# Monitor relayer logs
tail -f relayer/logs/relayer.log
```

## Monitoring & Maintenance

### 1. System Health Monitoring

Set up monitoring for:
- Contract state and balances
- Relayer network status
- Cross-chain message processing
- Gas price fluctuations
- Security alerts

### 2. Regular Maintenance Tasks

- **Weekly**: Review relayer performance metrics
- **Monthly**: Security audit of new integrations
- **Quarterly**: Update contract dependencies
- **As needed**: Emergency response procedures

### 3. Upgrade Procedures

For contract upgrades:
1. Deploy new contract versions
2. Update relayer configurations
3. Migrate critical state if necessary
4. Coordinate with partner protocols

## Security Checklist

### Pre-Production
- [ ] All contracts verified on block explorers
- [ ] Relayer private keys secured in hardware wallets
- [ ] Multi-signature setup for critical functions
- [ ] Emergency pause mechanisms tested
- [ ] Bug bounty program initiated

### Production
- [ ] Monitoring alerts configured
- [ ] Incident response plan documented
- [ ] Regular security audits scheduled
- [ ] Backup relayer infrastructure ready
- [ ] Communication channels established

## Troubleshooting

### Common Issues

1. **Deployment Failures**
   - Check gas prices and limits
   - Verify RPC endpoint connectivity
   - Ensure sufficient wallet balance

2. **Relayer Not Processing**
   - Check relayer authorization
   - Verify event emission on source chain
   - Monitor gas credit balance

3. **Cross-Chain Delays**
   - Check block confirmations
   - Verify relayer network consensus
   - Monitor destination chain congestion

### Support Resources

- **Technical Documentation**: [IU2U-Contracts](https://github.com/DINetworks/IU2U-Contracts)
- **Community Support**: Discord/Telegram channels
- **Developer Support**: Email support team
- **Emergency Contact**: 24/7 incident response team

## Production Checklist

### Before Mainnet Launch
- [ ] All testnets successfully deployed and tested
- [ ] Security audits completed and issues resolved
- [ ] Relayer infrastructure redundancy established
- [ ] Monitoring and alerting systems operational
- [ ] Documentation updated and reviewed
- [ ] Emergency procedures tested
- [ ] Community and partner communications prepared

### Post-Launch
- [ ] Monitor system health for first 48 hours
- [ ] Validate cross-chain operations
- [ ] Confirm DEX aggregation functionality
- [ ] Check meta-transaction processing
- [ ] Verify relayer network stability
- [ ] Document any issues and resolutions

---

*For the latest deployment scripts and configuration examples, refer to the [IU2U-Contracts repository](https://github.com/DINetworks/IU2U-Contracts).*
