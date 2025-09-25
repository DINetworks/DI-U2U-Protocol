# IU2U Protocol - Developer Integration Guide

## Overview

This guide provides comprehensive instructions for integrating IU2U Protocol into your decentralized applications (dApps), smart contracts, and frontend applications. IU2U enables cross-chain interoperability, gasless transactions, and advanced DEX aggregation.

## Quick Integration

### Prerequisites

- **Smart Contract Development**: Solidity ^0.8.20
- **Frontend Development**: JavaScript/TypeScript with ethers.js
- **Network Support**: EVM-compatible blockchains
- **Testing Framework**: Hardhat for smart contract testing

### Installation

```bash
# Install IU2U SDK (when available)
npm install @iu2u/sdk ethers

# For smart contract integration
npm install @openzeppelin/contracts
```

## Smart Contract Integration

### 1. Basic Cross-Chain Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@iu2u/contracts/interfaces/IIU2UGateway.sol";
import "@iu2u/contracts/IU2UExecutable.sol";

contract CrossChainMessenger is IU2UExecutable {
    struct Message {
        string content;
        address sender;
        string sourceChain;
        uint256 timestamp;
    }
    
    Message[] public messages;
    mapping(address => uint256) public messageCount;
    
    event MessageSent(
        string indexed destinationChain,
        address indexed sender,
        string content
    );
    
    event MessageReceived(
        string indexed sourceChain,
        address indexed sender,
        string content
    );
    
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
        
        emit MessageSent(destinationChain, msg.sender, message);
    }
    
    // Receive cross-chain message
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
        
        messageCount[_stringToAddress(sourceAddress)]++;
        
        emit MessageReceived(sourceChain, _stringToAddress(sourceAddress), messageContent);
    }
    
    // Handle cross-chain message with tokens
    function _executeWithToken(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload,
        string calldata symbol,
        uint256 amount
    ) internal override {
        // Process tokens (IU2U tokens are already minted to this contract)
        IERC20 iu2u = IERC20(gateway.getTokenAddress("IU2U"));
        
        // Decode payload and process with tokens
        (string memory messageContent, address recipient) = abi.decode(payload, (string, address));
        
        // Transfer tokens to recipient
        iu2u.transfer(recipient, amount);
        
        // Store message
        messages.push(Message({
            content: messageContent,
            sender: _stringToAddress(sourceAddress),
            sourceChain: sourceChain,
            timestamp: block.timestamp
        }));
        
        emit MessageReceived(sourceChain, _stringToAddress(sourceAddress), messageContent);
    }
    
    // Utility functions
    function _addressToString(address addr) internal pure returns (string memory) {
        return Strings.toHexString(uint160(addr), 20);
    }
    
    function _stringToAddress(string memory str) internal pure returns (address) {
        bytes memory data = bytes(str);
        uint160 addr = 0;
        
        for (uint i = 2; i < data.length; i++) {
            addr *= 16;
            uint8 b = uint8(data[i]);
            if (b >= 48 && b <= 57) addr += b - 48;
            else if (b >= 97 && b <= 102) addr += b - 87;
            else if (b >= 65 && b <= 70) addr += b - 55;
        }
        
        return address(addr);
    }
}
```

### 2. DeFi Protocol Integration

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@iu2u/contracts/interfaces/IIU2UGateway.sol";
import "@iu2u/contracts/IU2UExecutable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract CrossChainLendingProtocol is IU2UExecutable, ReentrancyGuard {
    struct LoanRequest {
        address borrower;
        uint256 amount;
        uint256 interestRate;
        uint256 duration;
        string collateralChain;
        bool fulfilled;
    }
    
    mapping(bytes32 => LoanRequest) public loanRequests;
    mapping(address => uint256) public deposits;
    
    event LoanRequested(
        bytes32 indexed loanId,
        address indexed borrower,
        uint256 amount,
        string collateralChain
    );
    
    event LoanFulfilled(
        bytes32 indexed loanId,
        address indexed lender
    );
    
    constructor(address gateway_) IU2UExecutable(gateway_) {}
    
    // Request loan with collateral on another chain
    function requestLoan(
        uint256 amount,
        uint256 interestRate,
        uint256 duration,
        string memory collateralChain,
        address collateralContract,
        uint256 collateralAmount
    ) external {
        bytes32 loanId = keccak256(abi.encodePacked(
            msg.sender,
            amount,
            block.timestamp
        ));
        
        loanRequests[loanId] = LoanRequest({
            borrower: msg.sender,
            amount: amount,
            interestRate: interestRate,
            duration: duration,
            collateralChain: collateralChain,
            fulfilled: false
        });
        
        // Send cross-chain call to lock collateral
        bytes memory payload = abi.encode(loanId, collateralAmount);
        gateway.callContract(
            collateralChain,
            _addressToString(collateralContract),
            payload
        );
        
        emit LoanRequested(loanId, msg.sender, amount, collateralChain);
    }
    
    // Fulfill loan request
    function fulfillLoan(bytes32 loanId) external nonReentrant {
        LoanRequest storage loan = loanRequests[loanId];
        require(!loan.fulfilled, "Loan already fulfilled");
        require(deposits[msg.sender] >= loan.amount, "Insufficient lender balance");
        
        loan.fulfilled = true;
        deposits[msg.sender] -= loan.amount;
        
        // Transfer loan amount to borrower
        IERC20 iu2u = IERC20(gateway.getTokenAddress("IU2U"));
        iu2u.transfer(loan.borrower, loan.amount);
        
        emit LoanFulfilled(loanId, msg.sender);
    }
    
    // Deposit funds as lender
    function deposit(uint256 amount) external {
        IERC20 iu2u = IERC20(gateway.getTokenAddress("IU2U"));
        iu2u.transferFrom(msg.sender, address(this), amount);
        deposits[msg.sender] += amount;
    }
    
    // Handle cross-chain collateral confirmation
    function _execute(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload
    ) internal override {
        (bytes32 loanId, bool collateralLocked) = abi.decode(payload, (bytes32, bool));
        
        if (collateralLocked) {
            // Collateral successfully locked, loan can proceed
            // Additional logic here
        }
    }
    
    function _executeWithToken(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload,
        string calldata symbol,
        uint256 amount
    ) internal override {
        // Handle loan repayment from another chain
        (bytes32 loanId) = abi.decode(payload, (bytes32));
        
        // Process loan repayment
        _processRepayment(loanId, amount);
    }
    
    function _processRepayment(bytes32 loanId, uint256 amount) internal {
        // Implementation for loan repayment processing
        // Update loan status, distribute interest, etc.
    }
    
    function _addressToString(address addr) internal pure returns (string memory) {
        return Strings.toHexString(uint160(addr), 20);
    }
}
```

### 3. DEX Integration with Aggregation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@iu2u/contracts/interfaces/ICrossChainAggregator.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract DeFiVault {
    ICrossChainAggregator public aggregator;
    mapping(address => mapping(address => uint256)) public userBalances;
    
    event SwapExecuted(
        address indexed user,
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 amountOut
    );
    
    constructor(address aggregator_) {
        aggregator = ICrossChainAggregator(aggregator_);
    }
    
    function deposit(address token, uint256 amount) external {
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        userBalances[msg.sender][token] += amount;
    }
    
    function autoRebalance(
        address tokenIn,
        address tokenOut,
        uint256 amount,
        uint256 minAmountOut
    ) external {
        require(userBalances[msg.sender][tokenIn] >= amount, "Insufficient balance");
        
        userBalances[msg.sender][tokenIn] -= amount;
        
        // Approve aggregator to spend tokens
        IERC20(tokenIn).approve(address(aggregator), amount);
        
        // Execute optimal swap through aggregator
        uint256 amountOut = aggregator.executeSwap(
            ICrossChainAggregator.SwapParams({
                tokenIn: tokenIn,
                tokenOut: tokenOut,
                amountIn: amount,
                minAmountOut: minAmountOut,
                routerType: 0, // Auto-select best router
                to: address(this),
                deadline: block.timestamp + 1800, // 30 minutes
                swapData: ""
            })
        );
        
        userBalances[msg.sender][tokenOut] += amountOut;
        
        emit SwapExecuted(msg.sender, tokenIn, tokenOut, amount, amountOut);
    }
    
    function withdraw(address token, uint256 amount) external {
        require(userBalances[msg.sender][token] >= amount, "Insufficient balance");
        
        userBalances[msg.sender][token] -= amount;
        IERC20(token).transfer(msg.sender, amount);
    }
}
```

## Frontend Integration

### 1. Basic Setup

```javascript
// Initialize IU2U provider
import { ethers } from 'ethers';

class IU2UProvider {
    constructor(config) {
        this.config = config;
        this.providers = {};
        this.contracts = {};
        this.signer = null;
        
        // Initialize providers for each chain
        for (const [chain, rpc] of Object.entries(config.rpcs)) {
            this.providers[chain] = new ethers.providers.JsonRpcProvider(rpc);
        }
    }
    
    async connect(privateKey) {
        this.signer = new ethers.Wallet(privateKey);
        
        // Connect signer to each provider
        for (const [chain, provider] of Object.entries(this.providers)) {
            this.providers[chain] = provider.connect(this.signer);
        }
    }
    
    async getContract(chain) {
        if (!this.contracts[chain]) {
            const address = this.config.contracts[chain];
            const provider = this.providers[chain];
            this.contracts[chain] = new ethers.Contract(address, IU2U_ABI, provider);
        }
        return this.contracts[chain];
    }
    
    // Deposit U2U for IU2U on U2U chain
    async deposit(amount) {
        const u2uContract = await this.getContract('u2u');
        const tx = await u2uContract.deposit({
            value: ethers.utils.parseEther(amount)
        });
        return await tx.wait();
    }
    
    // Send cross-chain transaction
    async callContract(destinationChain, contractAddress, payload) {
        const u2uContract = await this.getContract('u2u');
        const tx = await u2uContract.callContract(
            destinationChain,
            contractAddress,
            payload
        );
        return await tx.wait();
    }
    
    // Execute gasless transaction
    async executeMetaTransaction(metaTx, targetChain) {
        const nonce = await this.getNonce(metaTx.from, targetChain);
        const deadline = Math.floor(Date.now() / 1000) + 3600; // 1 hour
        
        const metaTxWithNonce = {
            ...metaTx,
            nonce,
            deadline
        };
        
        const signature = await this.signMetaTransaction(metaTxWithNonce, targetChain);
        
        return await this.submitMetaTransaction(metaTxWithNonce, signature, targetChain);
    }
    
    async getNonce(address, chain) {
        const contract = await this.getContract(chain);
        return await contract.getNonce(address);
    }
    
    async signMetaTransaction(metaTx, chain) {
        const contract = await this.getContract(chain);
        const domain = {
            name: "MetaTxGateway",
            version: "1",
            chainId: this.config.chainIds[chain],
            verifyingContract: contract.address
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
    
    async submitMetaTransaction(metaTx, signature, chain) {
        // Submit to relayer endpoint
        const response = await fetch(`${this.config.relayerUrl}/meta-tx/${chain}`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ metaTx, signature })
        });
        
        return await response.json();
    }
}

// Usage example
const iu2uProvider = new IU2UProvider({
    rpcs: {
        u2u: 'https://rpc.testnet.ms',
        ethereum: 'https://mainnet.infura.io/v3/KEY',
        bsc: 'https://bsc-dataseed1.binance.org'
    },
    contracts: {
        u2u: '0x...',
        ethereum: '0x...',
        bsc: '0x...'
    },
    chainIds: {
        u2u: 2484,
        ethereum: 1,
        bsc: 56
    },
    relayerUrl: 'http://localhost:3001'
});
```

### 2. React Hook for IU2U Integration

```javascript
// useIU2U.js
import { useState, useEffect, useCallback } from 'react';
import { ethers } from 'ethers';

export const useIU2U = (config) => {
    const [provider, setProvider] = useState(null);
    const [account, setAccount] = useState(null);
    const [balance, setBalance] = useState('0');
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);
    
    const connect = useCallback(async () => {
        try {
            setLoading(true);
            setError(null);
            
            if (window.ethereum) {
                const provider = new ethers.providers.Web3Provider(window.ethereum);
                await provider.send("eth_requestAccounts", []);
                const signer = provider.getSigner();
                const address = await signer.getAddress();
                
                setProvider(provider);
                setAccount(address);
                
                // Get IU2U balance
                const iu2uContract = new ethers.Contract(
                    config.contracts.iu2u,
                    ['function balanceOf(address) view returns (uint256)'],
                    provider
                );
                const balance = await iu2uContract.balanceOf(address);
                setBalance(ethers.utils.formatEther(balance));
            }
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    }, [config]);
    
    const sendCrossChain = useCallback(async (destinationChain, amount, recipient) => {
        if (!provider) throw new Error('Not connected');
        
        try {
            setLoading(true);
            const signer = provider.getSigner();
            const iu2uContract = new ethers.Contract(
                config.contracts.iu2u,
                [
                    'function sendToken(string memory destinationChain, string memory destinationAddress, string memory symbol, uint256 amount) external'
                ],
                signer
            );
            
            const tx = await iu2uContract.sendToken(
                destinationChain,
                recipient,
                'IU2U',
                ethers.utils.parseEther(amount)
            );
            
            return await tx.wait();
        } catch (err) {
            setError(err.message);
            throw err;
        } finally {
            setLoading(false);
        }
    }, [provider, config]);
    
    const executeGaslessTransaction = useCallback(async (targetContract, functionData, targetChain) => {
        if (!provider) throw new Error('Not connected');
        
        try {
            setLoading(true);
            const signer = provider.getSigner();
            const userAddress = await signer.getAddress();
            
            // Prepare meta-transaction
            const metaTx = {
                from: userAddress,
                to: targetContract,
                value: 0,
                data: functionData,
                nonce: 0, // Will be set by provider
                deadline: 0 // Will be set by provider
            };
            
            // Submit via relayer
            const response = await fetch(`${config.relayerUrl}/submit-meta-tx`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    metaTx,
                    chain: targetChain,
                    signer: userAddress
                })
            });
            
            return await response.json();
        } catch (err) {
            setError(err.message);
            throw err;
        } finally {
            setLoading(false);
        }
    }, [provider, config]);
    
    return {
        provider,
        account,
        balance,
        loading,
        error,
        connect,
        sendCrossChain,
        executeGaslessTransaction
    };
};

// Component usage
import React from 'react';
import { useIU2U } from './useIU2U';

const IU2UWallet = () => {
    const {
        account,
        balance,
        loading,
        error,
        connect,
        sendCrossChain
    } = useIU2U({
        contracts: {
            iu2u: '0x...' // IU2U contract address
        },
        relayerUrl: 'http://localhost:3001'
    });
    
    const handleSendCrossChain = async () => {
        try {
            await sendCrossChain('ethereum', '10', '0xRecipientAddress');
            alert('Cross-chain transfer initiated!');
        } catch (err) {
            alert('Transfer failed: ' + err.message);
        }
    };
    
    return (
        <div>
            <h2>IU2U Wallet</h2>
            {!account ? (
                <button onClick={connect} disabled={loading}>
                    {loading ? 'Connecting...' : 'Connect Wallet'}
                </button>
            ) : (
                <div>
                    <p>Account: {account}</p>
                    <p>IU2U Balance: {balance}</p>
                    <button onClick={handleSendCrossChain} disabled={loading}>
                        Send 10 IU2U to Ethereum
                    </button>
                </div>
            )}
            {error && <p style={{color: 'red'}}>Error: {error}</p>}
        </div>
    );
};

export default IU2UWallet;
```

## Testing Integration

### 1. Smart Contract Tests

```javascript
// test/CrossChainMessenger.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("CrossChainMessenger", function () {
    let crossChainMessenger;
    let mockGateway;
    let owner, user1, user2;
    
    beforeEach(async function () {
        [owner, user1, user2] = await ethers.getSigners();
        
        // Deploy mock gateway
        const MockGateway = await ethers.getContractFactory("MockIU2UGateway");
        mockGateway = await MockGateway.deploy();
        
        // Deploy messenger contract
        const CrossChainMessenger = await ethers.getContractFactory("CrossChainMessenger");
        crossChainMessenger = await CrossChainMessenger.deploy(mockGateway.address);
    });
    
    it("Should send cross-chain message", async function () {
        const destinationChain = "ethereum";
        const destinationContract = user2.address;
        const message = "Hello cross-chain!";
        
        await expect(
            crossChainMessenger.connect(user1).sendMessage(
                destinationChain,
                destinationContract,
                message
            )
        ).to.emit(crossChainMessenger, "MessageSent")
         .withArgs(destinationChain, user1.address, message);
    });
    
    it("Should receive cross-chain message", async function () {
        const sourceChain = "bsc";
        const sourceAddress = user1.address.toLowerCase();
        const message = "Hello from BSC!";
        const payload = ethers.utils.defaultAbiCoder.encode(["string"], [message]);
        
        // Simulate gateway call
        await mockGateway.simulateExecute(
            crossChainMessenger.address,
            sourceChain,
            sourceAddress,
            payload
        );
        
        const messageCount = await crossChainMessenger.messageCount(user1.address);
        expect(messageCount).to.equal(1);
        
        const storedMessage = await crossChainMessenger.messages(0);
        expect(storedMessage.content).to.equal(message);
        expect(storedMessage.sourceChain).to.equal(sourceChain);
    });
});
```

### 2. Integration Tests

```javascript
// test/integration/CrossChainFlow.test.js
describe("Cross-Chain Integration", function () {
    let u2uGateway, ethereumGateway;
    let u2uProvider, ethereumProvider;
    
    before(async function () {
        // Setup providers for different chains
        u2uProvider = new ethers.providers.JsonRpcProvider("http://localhost:8545");
        ethereumProvider = new ethers.providers.JsonRpcProvider("http://localhost:8546");
        
        // Deploy contracts on both chains
        u2uGateway = await deployGateway(u2uProvider);
        ethereumGateway = await deployGateway(ethereumProvider);
    });
    
    it("Should execute end-to-end cross-chain transaction", async function () {
        // 1. Deposit U2U on U2U
        const depositTx = await u2uGateway.deposit({
            value: ethers.utils.parseEther("100")
        });
        await depositTx.wait();
        
        // 2. Send cross-chain call to Ethereum
        const payload = ethers.utils.defaultAbiCoder.encode(
            ["string"],
            ["Hello Ethereum!"]
        );
        
        const callTx = await u2uGateway.callContract(
            "ethereum",
            ethereumGateway.address,
            payload
        );
        await callTx.wait();
        
        // 3. Wait for relayer to process (in real scenario)
        // Here we simulate the relayer execution
        
        // 4. Verify execution on Ethereum
        const executed = await ethereumGateway.isCommandExecuted(commandId);
        expect(executed).to.be.true;
    });
});
```

## Best Practices

### 1. Security Considerations

- **Input Validation**: Always validate cross-chain payloads
- **Reentrancy Protection**: Use OpenZeppelin's ReentrancyGuard
- **Access Control**: Implement proper role-based access control
- **Gas Limits**: Set appropriate gas limits for cross-chain calls

```solidity
// Example security implementation
contract SecureContract is IU2UExecutable, ReentrancyGuard, AccessControl {
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    
    modifier validChain(string memory chain) {
        require(supportedChains[chain], "Unsupported chain");
        _;
    }
    
    function _execute(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload
    ) internal override nonReentrant validChain(sourceChain) {
        // Validate source address
        require(trustedSources[sourceChain][sourceAddress], "Untrusted source");
        
        // Validate payload size
        require(payload.length <= MAX_PAYLOAD_SIZE, "Payload too large");
        
        // Process securely
        _processPayload(payload);
    }
}
```

### 2. Gas Optimization

- **Batch Operations**: Group multiple operations when possible
- **Efficient Encoding**: Use packed encoding for payloads
- **State Management**: Minimize storage operations

```solidity
// Gas-optimized batch processing
function batchSendMessages(
    string[] memory destinationChains,
    address[] memory destinationContracts,
    string[] memory messages
) external {
    require(
        destinationChains.length == destinationContracts.length &&
        destinationContracts.length == messages.length,
        "Array length mismatch"
    );
    
    for (uint i = 0; i < destinationChains.length; i++) {
        bytes memory payload = abi.encode(messages[i]);
        gateway.callContract(
            destinationChains[i],
            _addressToString(destinationContracts[i]),
            payload
        );
    }
}
```

### 3. Error Handling

- **Graceful Failures**: Handle cross-chain failures gracefully
- **Retry Mechanisms**: Implement retry logic for failed operations
- **Event Logging**: Comprehensive event logging for debugging

```solidity
contract RobustContract is IU2UExecutable {
    mapping(bytes32 => bool) public failedExecutions;
    
    function _execute(
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload
    ) internal override {
        bytes32 executionId = keccak256(abi.encodePacked(
            sourceChain,
            sourceAddress,
            payload,
            block.timestamp
        ));
        
        try this._processPayload(payload) {
            emit ExecutionSuccess(executionId);
        } catch Error(string memory reason) {
            failedExecutions[executionId] = true;
            emit ExecutionFailed(executionId, reason);
        } catch {
            failedExecutions[executionId] = true;
            emit ExecutionFailed(executionId, "Unknown error");
        }
    }
    
    function retryExecution(bytes32 executionId, bytes calldata payload) external {
        require(failedExecutions[executionId], "Execution not failed");
        
        failedExecutions[executionId] = false;
        _processPayload(payload);
    }
}
```

## Support and Resources

### Documentation
- **Technical Specifications**: [TECHNICAL_SPECIFICATIONS.md](./TECHNICAL_SPECIFICATIONS.md)
- **Deployment Guide**: [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md)
- **Contract Repository**: [IU2U-Contracts](https://github.com/DINetworks/IU2U-Contracts)

### Community
- **Developer Discord**: Join for real-time support
- **GitHub Issues**: Report bugs and request features
- **Developer Email**: technical-support@iu2u.com

### Tools and Libraries
- **Hardhat Plugin**: `@iu2u/hardhat-plugin`
- **Testing Utilities**: `@iu2u/test-utils`
- **Frontend SDK**: `@iu2u/sdk`

---

*This integration guide is continuously updated. For the latest examples and best practices, refer to the [IU2U-Contracts repository](https://github.com/DINetworks/IU2U-Contracts).*
