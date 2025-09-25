# IU2U Protocol - Technical Specifications

## Overview

The IU2U (Interoperable U2U) Protocol is a comprehensive cross-chain infrastructure that enables seamless asset transfers, DEX aggregation, and gasless transactions across multiple EVM-compatible blockchains. This document provides detailed technical specifications based on the live implementation in the [IU2U-Contracts repository](https://github.com/DINetworks/IU2U-Contracts).

## Architecture Components

### 1. Core Gateway Contract (`IU2U.sol`)

The main gateway contract handles cross-chain message passing and U2U↔IU2U conversion.

#### Key Functions

```solidity
// U2U ↔ IU2U Conversion (U2U Chain Only)
function deposit() public payable onlyCrossfiChain
function withdraw(uint256 amount_) public onlyCrossfiChain

// Cross-Chain Operations
function callContract(
    string memory destinationChain,
    string memory destinationContractAddress,
    bytes memory payload
) external

function callContractWithToken(
    string memory destinationChain,
    string memory destinationContractAddress,
    bytes memory payload,
    string memory symbol,
    uint256 amount
) external

function sendToken(
    string memory destinationChain,
    string memory destinationAddress,
    string memory symbol,
    uint256 amount
) external

// Command Execution (Relayer Only)
function execute(
    bytes32 commandId,
    Command[] memory commands,
    bytes memory signature
) external onlyRelayer
```

#### State Variables

```solidity
uint256 u2u_chainid = 2484; // 39 for mainnet
mapping(address => bool) public whitelisted; // Whitelisted relayers
mapping(bytes32 => bool) public commandExecuted; // Executed commands
mapping(string => uint256) public chainIds; // Supported chains
mapping(bytes32 => bytes) public approvedPayloads; // Approved payloads
```

#### Events

```solidity
event ContractCall(
    address indexed sender,
    string destinationChain,
    string destinationContractAddress,
    bytes32 indexed payloadHash,
    bytes payload
);

event ContractCallWithToken(
    address indexed sender,
    string destinationChain,
    string destinationContractAddress,
    bytes32 indexed payloadHash,
    bytes payload,
    string symbol,
    uint256 amount
);

event TokenSent(
    address indexed sender,
    string destinationChain,
    string destinationAddress,
    string symbol,
    uint256 amount
);

event Deposited(address indexed user, uint256 amount);
event Withdrawn(address indexed user, uint256 amount);
```

### 2. Meta-Transaction System

#### MetaTxGasCreditVault (`MetaTxGasCreditVault.sol`)

Manages IU2U-based gas credits on U2U chain with DIA Oracle integration.

```solidity
// Core Functions
function depositCredits(uint256 amount) external
function withdrawCredits(uint256 amount) external
function consumeCredits(address user, uint256 gasUsed, uint256 gasPrice) external
function getCreditBalance(address user) external view returns (uint256)

// Oracle Integration
function updateGasPrice(uint256 chainId, uint256 gasPrice) external onlyOracle
function getConversionRate(uint256 chainId) external view returns (uint256)
```

#### MetaTxGateway (`MetaTxGateway.sol`)

Handles meta-transaction execution on destination chains.

```solidity
struct MetaTransaction {
    address from;      // User who signed the transaction
    address to;        // Target contract to call
    uint256 value;     // ETH value to send (usually 0)
    bytes data;        // Function call data
    uint256 nonce;     // User's current nonce
    uint256 deadline;  // Transaction deadline
}

// Core Functions
function executeMetaTransaction(
    MetaTransaction memory metaTx,
    bytes memory signature
) external returns (bool)

function batchExecuteMetaTransactions(
    MetaTransaction[] memory metaTxs,
    bytes[] memory signatures
) external returns (bool[] memory)

// EIP-712 Functions
function getDomainSeparator() public view returns (bytes32)
function getMetaTransactionHash(MetaTransaction memory metaTx) public view returns (bytes32)
function getNonce(address user) external view returns (uint256)
```

### 3. Cross-Chain DEX Aggregation

#### CrossChainAggregator (`CrossChainAggregator.sol`)

Multi-protocol DEX aggregation system supporting 37+ protocols.

```solidity
struct SwapParams {
    address tokenIn;
    address tokenOut;
    uint256 amountIn;
    uint256 minAmountOut;
    uint8 routerType;
    address to;
    uint256 deadline;
    bytes swapData;
}

// Core Swap Functions
function executeSwap(SwapParams memory params) external payable returns (uint256)
function executeMultiSwap(SwapParams[] memory swaps) external payable returns (uint256[] memory)
function getQuote(SwapParams memory params) external view returns (uint256)
```

#### Supported DEX Protocols

| Router Type | Protocol | Chains | Liquidity Type |
|-------------|----------|--------|----------------|
| 0-3 | Uniswap V2/V3 | Ethereum, Polygon, Arbitrum | AMM + Concentrated |
| 4-7 | SushiSwap V2/V3 | Multi-chain | AMM + Concentrated |
| 8-11 | PancakeSwap V2/V3 | BSC, Ethereum | AMM + Concentrated |
| 12-15 | Curve Finance | Multi-chain | Stableswap |
| 16-19 | Balancer V2 | Ethereum, Polygon | Weighted Pools |
| 20-23 | TraderJoe | Avalanche | AMM |
| 24-27 | QuickSwap | Polygon | AMM |
| 28-31 | SpookySwap | Fantom | AMM |
| 32-36 | Others | Various | Specialized |

### 4. IU2UExecutable Base Contract

Abstract contract for dApps wanting to receive cross-chain calls.

```solidity
abstract contract IU2UExecutable {
    IIU2UGateway public immutable gateway;
    
    constructor(address gateway_) {
        gateway = IIU2UGateway(gateway_);
    }
    
    modifier onlyGateway() {
        require(msg.sender == address(gateway), "Only gateway");
        _;
    }
    
    // Implemented functions
    function execute(
        bytes32 commandId,
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload
    ) external override onlyGateway;
    
    function executeWithToken(
        bytes32 commandId,
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload,
        string calldata symbol,
        uint256 amount
    ) external override onlyGateway;
    
    // Abstract functions to implement
    function _execute(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload
    ) internal virtual;
    
    function _executeWithToken(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload,
        string calldata symbol,
        uint256 amount
    ) internal virtual;
}
```

## Command System

### Command Types

```solidity
uint256 public constant COMMAND_APPROVE_CONTRACT_CALL = 0;
uint256 public constant COMMAND_APPROVE_CONTRACT_CALL_WITH_MINT = 1;
uint256 public constant COMMAND_BURN_TOKEN = 2;
uint256 public constant COMMAND_MINT_TOKEN = 4;
```

### Command Structure

```solidity
struct Command {
    uint256 commandType;
    bytes data;
}
```

## Security Model

### Multi-Signature Relayer Network

- **Whitelisted Relayers**: Only authorized relayers can execute commands
- **Command Validation**: All commands must be cryptographically signed
- **Replay Protection**: Commands can only be executed once via `commandExecuted` mapping
- **Payload Verification**: Payload hashes are verified before execution

### EIP-712 Signature Standard

Meta-transactions use EIP-712 for structured data signing:

```solidity
bytes32 private constant META_TRANSACTION_TYPEHASH = keccak256(
    "MetaTransaction(address from,address to,uint256 value,bytes data,uint256 nonce,uint256 deadline)"
);
```

## Gas Credit System

### Credit Calculation

```
gasUsd = gasUsed * gasPrice * nativeTokenPrice
creditsNeeded = gasUsd (in cents)
```

### DIA Oracle Integration

- Real-time price feeds for accurate gas cost calculation
- Multi-chain gas price tracking
- Automated conversion rate updates

## Network Configuration

### Supported Chains

| Chain | Chain ID | Native Token | Status |
|-------|----------|--------------|--------|
| U2U | 2484/39 | U2U | Native (Source) |
| Ethereum | 1 | ETH | Supported |
| BSC | 56 | BNB | Supported |
| Polygon | 137 | MATIC | Supported |
| Avalanche | 43114 | AVAX | Supported |
| Arbitrum | 42161 | ETH | Supported |
| Optimism | 10 | ETH | Supported |
| Base | 8453 | ETH | Supported |

### Deployment Requirements

| Chain | Estimated Gas Cost | Purpose |
|-------|-------------------|---------|
| U2U | 0.1 U2U | Core contracts deployment |
| Ethereum | 0.05 ETH | Gateway and vault deployment |
| BSC | 0.01 BNB | Gateway deployment |
| Polygon | 50 MATIC | Gateway deployment |

## Integration Patterns

### 1. DeFi Protocol Integration

```solidity
contract MyDeFiProtocol is IU2UExecutable {
    constructor(address gateway_) IU2UExecutable(gateway_) {}
    
    function _execute(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload
    ) internal override {
        // Decode payload
        (uint256 amount, address recipient) = abi.decode(payload, (uint256, address));
        
        // Process DeFi operation
        processLending(amount, recipient);
    }
    
    function _executeWithToken(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload,
        string calldata symbol,
        uint256 amount
    ) internal override {
        // IU2U tokens are pre-minted to this contract
        // Process cross-chain operation with tokens
        processWithTokens(amount, payload);
    }
}
```

### 2. Cross-Chain Bridge

```solidity
contract CrossChainBridge {
    IIU2UGateway public iu2uGateway;
    
    function bridgeTokens(
        string memory destinationChain,
        address destinationContract,
        uint256 amount,
        bytes memory bridgeData
    ) external {
        // Burn tokens on source chain
        burnTokens(msg.sender, amount);
        
        // Send cross-chain call to mint on destination
        iu2uGateway.callContractWithToken(
            destinationChain,
            addressToString(destinationContract),
            bridgeData,
            "IU2U",
            amount
        );
    }
}
```

### 3. Meta-Transaction Integration

```javascript
// Frontend meta-transaction example
const metaTx = {
    from: userAddress,
    to: contractAddress,
    value: 0,
    data: encodedFunctionCall,
    nonce: await iu2u.getNonce(userAddress),
    deadline: Date.now() + 3600000 // 1 hour
};

// Sign with EIP-712
const signature = await signer._signTypedData(domain, types, metaTx);

// Submit via relayer
await iu2u.submitMetaTransaction(metaTx, signature, 'ethereum');
```

## Testing Framework

### Unit Tests

- Complete test coverage for all core contracts
- Mock contracts for external dependencies
- Gas optimization testing

### Integration Tests

- Cross-chain message passing simulation
- Multi-chain DEX aggregation testing
- Meta-transaction flow validation

### Deployment Tests

- Automated deployment verification
- Contract address validation
- Cross-chain configuration testing

## Performance Metrics

### Transaction Throughput

- **Cross-Chain Latency**: 2-5 minutes (depending on chain finality)
- **DEX Aggregation**: <30 seconds for optimal route calculation
- **Meta-Transaction Processing**: <1 minute execution time

### Gas Optimization

- **Batch Operations**: Up to 70% gas savings
- **Optimal Routing**: 15-30% better swap rates
- **Meta-Transaction Overhead**: <20% additional gas cost

## Monitoring & Maintenance

### Health Checks

- Relayer network status monitoring
- Contract state validation
- Cross-chain message verification

### Upgrade Mechanisms

- Proxy contract patterns for upgradability
- Timelock governance for critical changes
- Emergency pause functionality

## Security Audits

- **Smart Contract Audits**: Quarterly security reviews
- **Code Coverage**: >95% test coverage
- **Bug Bounty Program**: Active security researcher engagement
- **Formal Verification**: Core contract mathematical proofs

---

*For the most up-to-date technical implementation details, please refer to the [IU2U-Contracts repository](https://github.com/DINetworks/IU2U-Contracts).*
