# DI U2U Protocol - API Reference

## Overview

This document provides a comprehensive API reference for the IU2U Protocol smart contracts. The IU2U Protocol enables cross-chain interoperability, DEX aggregation, and gasless transactions across multiple EVM-compatible blockchains.

## Contract Interfaces

### IIU2UGateway Interface

The main interface for cross-chain operations and token management.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IIU2UGateway {
    // Events
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

    // Core Functions

    /**
     * @dev Deposit native U2U to mint equivalent IU2U tokens (1:1 ratio)
     * @notice Only available on U2U chain
     */
    function deposit() external payable;

    /**
     * @dev Withdraw U2U by burning IU2U tokens (1:1 ratio)
     * @param amount Amount of IU2U to burn for U2U
     * @notice Only available on U2U chain
     */
    function withdraw(uint256 amount) external;

    /**
     * @dev Execute a cross-chain contract call
     * @param destinationChain Name of the destination blockchain
     * @param destinationContractAddress Target contract address
     * @param payload Encoded function call data
     */
    function callContract(
        string memory destinationChain,
        string memory destinationContractAddress,
        bytes memory payload
    ) external;

    /**
     * @dev Execute a cross-chain contract call with token transfer
     * @param destinationChain Name of the destination blockchain
     * @param destinationContractAddress Target contract address
     * @param payload Encoded function call data
     * @param symbol Token symbol (must be "IU2U")
     * @param amount Amount of tokens to transfer
     */
    function callContractWithToken(
        string memory destinationChain,
        string memory destinationContractAddress,
        bytes memory payload,
        string memory symbol,
        uint256 amount
    ) external;

    /**
     * @dev Send tokens to another chain
     * @param destinationChain Name of the destination blockchain
     * @param destinationAddress Recipient address on destination chain
     * @param symbol Token symbol (must be "IU2U")
     * @param amount Amount of tokens to send
     */
    function sendToken(
        string memory destinationChain,
        string memory destinationAddress,
        string memory symbol,
        uint256 amount
    ) external;

    // View Functions

    /**
     * @dev Check if IU2U tokens are fully backed by U2U
     * @return bool True if fully backed
     */
    function isFullyBacked() external view returns (bool);

    /**
     * @dev Get the U2U balance held by the contract
     * @return uint256 U2U balance
     */
    function getU2UBalance() external view returns (uint256);

    /**
     * @dev Check if a command has been executed
     * @param commandId Unique command identifier
     * @return bool True if command has been executed
     */
    function isCommandExecuted(bytes32 commandId) external view returns (bool);

    /**
     * @dev Get all authorized relayers
     * @return address[] Array of relayer addresses
     */
    function getAllRelayers() external view returns (address[] memory);

    /**
     * @dev Get the number of authorized relayers
     * @return uint256 Number of relayers
     */
    function getRelayerCount() external view returns (uint256);

    // Chain Management

    /**
     * @dev Add a new supported chain
     * @param chainName Name of the chain
     * @param chainId Chain ID
     */
    function addChain(string calldata chainName, uint256 chainId) external;

    /**
     * @dev Remove a supported chain
     * @param chainName Name of the chain to remove
     */
    function removeChain(string calldata chainName) external;

    /**
     * @dev Get chain ID by name
     * @param chainName Name of the chain
     * @return uint256 Chain ID
     */
    function getChainId(string calldata chainName) external view returns (uint256);

    /**
     * @dev Get chain name by ID
     * @param chainId Chain ID
     * @return string Chain name
     */
    function getChainName(uint256 chainId) external view returns (string memory);

    // Admin Functions

    /**
     * @dev Add a whitelisted relayer
     * @param relayer Address of the relayer
     */
    function addWhitelistedRelayer(address relayer) external;

    /**
     * @dev Remove a whitelisted relayer
     * @param relayer Address of the relayer
     */
    function removeWhitelistedRelayer(address relayer) external;
}
```

### ICrossChainAggregator Interface

Interface for DEX aggregation and optimal routing across multiple protocols.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ICrossChainAggregator {
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

    event SwapExecuted(
        address indexed user,
        address indexed tokenIn,
        address indexed tokenOut,
        uint256 amountIn,
        uint256 amountOut,
        uint8 routerType
    );

    /**
     * @dev Execute a token swap through optimal DEX routing
     * @param params Swap parameters struct
     * @return amountOut Amount of tokens received
     */
    function executeSwap(SwapParams memory params) external payable returns (uint256 amountOut);

    /**
     * @dev Execute multiple swaps in batch
     * @param swaps Array of swap parameters
     * @return amountsOut Array of output amounts
     */
    function executeMultiSwap(SwapParams[] memory swaps) external payable returns (uint256[] memory amountsOut);

    /**
     * @dev Get quote for a potential swap
     * @param params Swap parameters
     * @return amountOut Expected output amount
     */
    function getQuote(SwapParams memory params) external view returns (uint256 amountOut);

    /**
     * @dev Get the best route for a swap across multiple DEXes
     * @param tokenIn Input token address
     * @param tokenOut Output token address
     * @param amountIn Input amount
     * @return routerType Best router type
     * @return amountOut Expected output amount
     */
    function getBestRoute(
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) external view returns (uint8 routerType, uint256 amountOut);
}
```

### IMetaTxGateway Interface

Interface for gasless transaction execution using meta-transactions.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IMetaTxGateway {
    struct MetaTransaction {
        address from;
        address to;
        uint256 value;
        bytes data;
        uint256 nonce;
        uint256 deadline;
    }

    event MetaTransactionExecuted(
        address indexed user,
        address indexed relayer,
        bytes32 indexed hash,
        bool success
    );

    /**
     * @dev Execute a meta-transaction (gasless transaction)
     * @param metaTx Meta-transaction struct
     * @param signature User's signature
     * @return success True if execution successful
     */
    function executeMetaTransaction(
        MetaTransaction memory metaTx,
        bytes memory signature
    ) external returns (bool success);

    /**
     * @dev Execute multiple meta-transactions in batch
     * @param metaTxs Array of meta-transactions
     * @param signatures Array of corresponding signatures
     * @return results Array of execution results
     */
    function batchExecuteMetaTransactions(
        MetaTransaction[] memory metaTxs,
        bytes[] memory signatures
    ) external returns (bool[] memory results);

    /**
     * @dev Get the current nonce for a user
     * @param user User address
     * @return nonce Current nonce
     */
    function getNonce(address user) external view returns (uint256 nonce);

    /**
     * @dev Get the domain separator for EIP-712
     * @return bytes32 Domain separator
     */
    function getDomainSeparator() external view returns (bytes32);

    /**
     * @dev Get the hash of a meta-transaction
     * @param metaTx Meta-transaction struct
     * @return bytes32 Transaction hash
     */
    function getMetaTransactionHash(MetaTransaction memory metaTx) external view returns (bytes32);

    /**
     * @dev Set relayer authorization status
     * @param relayer Relayer address
     * @param authorized Authorization status
     */
    function setRelayerAuthorization(address relayer, bool authorized) external;
}
```

### IU2UExecutable Abstract Contract

Base contract for dApps that want to receive cross-chain calls.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

abstract contract IU2UExecutable {
    IIU2UGateway public immutable gateway;

    error NotGateway();
    error InvalidAddress();

    constructor(address gateway_) {
        if (gateway_ == address(0)) revert InvalidAddress();
        gateway = IIU2UGateway(gateway_);
    }

    modifier onlyGateway() {
        if (msg.sender != address(gateway)) revert NotGateway();
        _;
    }

    /**
     * @dev Execute a cross-chain contract call
     * @param commandId Unique identifier for the command
     * @param sourceChain Name of the source chain
     * @param sourceAddress Address of the sender on the source chain
     * @param payload Call data
     */
    function execute(
        bytes32 commandId,
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload
    ) external onlyGateway {
        _execute(sourceChain, sourceAddress, payload);
    }

    /**
     * @dev Execute a cross-chain contract call with token transfer
     * @param commandId Unique identifier for the command
     * @param sourceChain Name of the source chain
     * @param sourceAddress Address of the sender on the source chain
     * @param payload Call data
     * @param symbol Token symbol
     * @param amount Token amount
     */
    function executeWithToken(
        bytes32 commandId,
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload,
        string calldata symbol,
        uint256 amount
    ) external onlyGateway {
        _executeWithToken(sourceChain, sourceAddress, payload, symbol, amount);
    }

    /**
     * @dev Internal function to handle cross-chain calls (must be implemented)
     * @param sourceChain Name of the source chain
     * @param sourceAddress Address of the sender on the source chain
     * @param payload Call data
     */
    function _execute(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload
    ) internal virtual;

    /**
     * @dev Internal function to handle cross-chain calls with tokens (must be implemented)
     * @param sourceChain Name of the source chain
     * @param sourceAddress Address of the sender on the source chain
     * @param payload Call data
     * @param symbol Token symbol
     * @param amount Token amount
     */
    function _executeWithToken(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload,
        string calldata symbol,
        uint256 amount
    ) internal virtual;
}
```

## Usage Examples

### Basic Cross-Chain Message Passing

```solidity
// Deploy on both source and destination chains
contract CrossChainMessenger is IU2UExecutable {
    struct Message {
        string content;
        address sender;
        string sourceChain;
        uint256 timestamp;
    }
    
    Message[] public messages;
    
    constructor(address gateway_) IU2UExecutable(gateway_) {}
    
    // Send message to another chain
    function sendMessage(
        string memory destinationChain,
        address destinationContract,
        string memory message
    ) external {
        bytes memory payload = abi.encode(message);
        gateway.callContract(
            destinationChain,
            _addressToString(destinationContract),
            payload
        );
    }
    
    // Receive message from another chain
    function _execute(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload
    ) internal override {
        (string memory messageContent) = abi.decode(payload, (string));
        
        messages.push(Message({
            content: messageContent,
            sender: _stringToAddress(sourceAddress),
            sourceChain: sourceChain,
            timestamp: block.timestamp
        }));
    }
    
    function _executeWithToken(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload,
        string calldata symbol,
        uint256 amount
    ) internal override {
        // Handle messages with token transfers
        // IU2U tokens are automatically minted to this contract
    }
}
```

### Cross-Chain DEX Integration

```solidity
contract CrossChainDEX {
    ICrossChainAggregator public aggregator;
    
    constructor(address aggregator_) {
        aggregator = ICrossChainAggregator(aggregator_);
    }
    
    function executeOptimalSwap(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 minAmountOut
    ) external returns (uint256 amountOut) {
        // Get the best route
        (uint8 routerType, uint256 expectedOut) = aggregator.getBestRoute(
            tokenIn,
            tokenOut,
            amountIn
        );
        
        require(expectedOut >= minAmountOut, "Insufficient output amount");
        
        // Execute swap with optimal router
        return aggregator.executeSwap(
            ICrossChainAggregator.SwapParams({
                tokenIn: tokenIn,
                tokenOut: tokenOut,
                amountIn: amountIn,
                minAmountOut: minAmountOut,
                routerType: routerType,
                to: msg.sender,
                deadline: block.timestamp + 1800,
                swapData: ""
            })
        );
    }
}
```

### Gasless Transaction Implementation

```javascript
// Frontend implementation for gasless transactions
class MetaTxProvider {
    constructor(gatewayAddress, signer, relayerUrl) {
        this.gateway = new ethers.Contract(gatewayAddress, META_TX_ABI, signer);
        this.signer = signer;
        this.relayerUrl = relayerUrl;
    }
    
    async executeGaslessTransaction(to, data, value = 0) {
        const userAddress = await this.signer.getAddress();
        const nonce = await this.gateway.getNonce(userAddress);
        const deadline = Math.floor(Date.now() / 1000) + 3600; // 1 hour
        
        const metaTx = {
            from: userAddress,
            to: to,
            value: value,
            data: data,
            nonce: nonce,
            deadline: deadline
        };
        
        // Sign the meta-transaction
        const signature = await this.signMetaTransaction(metaTx);
        
        // Submit to relayer
        const response = await fetch(`${this.relayerUrl}/execute`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ metaTx, signature })
        });
        
        return await response.json();
    }
    
    async signMetaTransaction(metaTx) {
        const domain = {
            name: "MetaTxGateway",
            version: "1",
            chainId: await this.signer.getChainId(),
            verifyingContract: this.gateway.address
        };
        
        const types = {
            MetaTransaction: [
                { name: "from", type: "address" },
                { name: "to", type: "address" },
                { name: "value", type: "uint256" },
                { name: "data", type: "bytes" },
                { name: "nonce", type: "uint256" },
                { name: "deadline", type: "uint256" }
            ]
        };
        
        return await this.signer._signTypedData(domain, types, metaTx);
    }
}
```

## Error Codes

### Common Errors

```solidity
// IU2U Gateway Errors
error ZeroValue();                    // When msg.value is 0 in deposit
error NotU2UChain();             // When calling U2U-only functions on other chains
error CallerNotWhitelistedRelayer(); // When non-relayer tries to execute commands
error CommandAlreadyExecuted();      // When trying to execute the same command twice
error InvalidDestinationChain();     // When destination chain is not supported
error InvalidTokenSymbol();          // When token symbol is not "IU2U"
error InsufficientBalance();         // When user has insufficient token balance

// Meta-Transaction Errors
error InvalidSignature();            // When meta-transaction signature is invalid
error DeadlineExpired();             // When meta-transaction deadline has passed
error InvalidNonce();                // When nonce is incorrect or replay attack
error UnauthorizedRelayer();         // When relayer is not authorized
error ExecutionFailed();             // When meta-transaction execution fails

// DEX Aggregator Errors
error UnsupportedRouter();           // When router type is not supported
error InsufficientOutputAmount();    // When swap output is below minimum
error DeadlineExceeded();            // When swap deadline is exceeded
error InvalidSwapPath();             // When token swap path is invalid
```

## Events Reference

### Core Events

```solidity
// Cross-chain message events
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

// Token transfer events
event TokenSent(
    address indexed sender,
    string destinationChain,
    string destinationAddress,
    string symbol,
    uint256 amount
);

// U2U conversion events
event Deposited(address indexed user, uint256 amount);
event Withdrawn(address indexed user, uint256 amount);

// Command execution events
event ContractCallApproved(
    bytes32 indexed commandId,
    string sourceChain,
    string sourceAddress,
    address indexed contractAddress,
    bytes32 indexed payloadHash
);

// Meta-transaction events
event MetaTransactionExecuted(
    address indexed user,
    address indexed relayer,
    bytes32 indexed hash,
    bool success
);

// DEX aggregation events
event SwapExecuted(
    address indexed user,
    address indexed tokenIn,
    address indexed tokenOut,
    uint256 amountIn,
    uint256 amountOut,
    uint8 routerType
);
```

## Supported DEX Protocols

### Router Types

| Router ID | Protocol | Chains | Type |
|-----------|----------|--------|------|
| 0-3 | Uniswap V2/V3 | Ethereum, Polygon, Arbitrum | AMM + Concentrated |
| 4-7 | SushiSwap V2/V3 | Multi-chain | AMM + Concentrated |
| 8-11 | PancakeSwap V2/V3 | BSC, Ethereum | AMM + Concentrated |
| 12-15 | Curve Finance | Multi-chain | Stableswap |
| 16-19 | Balancer V2 | Ethereum, Polygon | Weighted Pools |
| 20-23 | TraderJoe | Avalanche | AMM |
| 24-27 | QuickSwap | Polygon | AMM |
| 28-31 | SpookySwap | Fantom | AMM |
| 32-36 | Others | Various | Specialized |

## Network Configuration

### Supported Chains

```solidity
// Chain IDs and names
mapping(string => uint256) public chainIds = {
    "u2u": 2484,      // 39 for mainnet
    "ethereum": 1,
    "bsc": 56,
    "polygon": 137,
    "avalanche": 43114,
    "arbitrum": 42161,
    "optimism": 10,
    "base": 8453
};
```

### Contract Addresses

Contract addresses are deployment-specific. Refer to the [deployment files](https://github.com/DINetworks/IU2U-Contracts/tree/main/deployments) in the IU2U-Contracts repository for current addresses.

## Rate Limits and Gas Costs

### Gas Optimization

- **Batch Operations**: Use batch functions when processing multiple operations
- **Payload Size**: Keep cross-chain payloads under 32KB for efficiency
- **Router Selection**: Optimal router selection can save 15-30% on swap costs

### Recommended Limits

- **Cross-Chain Calls**: Maximum 10 calls per transaction
- **Meta-Transaction Batch**: Maximum 20 transactions per batch
- **DEX Swap Batch**: Maximum 5 swaps per batch

---

*For the latest API updates and additional examples, refer to the [IU2U-Contracts repository](https://github.com/DINetworks/IU2U-Contracts).*
