# Capítulo 13: Front-end Integration - Ethers.js e Web3

> **Para Desenvolvedores Experientes**: Se você já construiu SPAs com React/Vue, integrar blockchain é como adicionar mais uma API - mas assíncrona, com wallet authentication, e onde cada "POST request" custa dinheiro real. Esqueça `localStorage` para auth - agora é cryptographic signatures. Esqueça `fetch()` para dados - agora é leitura direto da blockchain.

---

## Índice
- [13.1 Web2 vs Web3 Frontend](#131-web2-vs-web3-frontend)
- [13.2 Ethers.js vs Web3.js](#132-ethersjs-vs-web3js)
- [13.3 Conectar Wallet](#133-conectar-wallet)
- [13.4 Ler Dados da Blockchain](#134-ler-dados-da-blockchain)
- [13.5 Enviar Transações](#135-enviar-transações)
- [13.6 Escutar Eventos](#136-escutar-eventos)
- [13.7 React Hooks para Web3](#137-react-hooks-para-web3)
- [13.8 Error Handling](#138-error-handling)
- [13.9 Boas Práticas de UX](#139-boas-práticas-de-ux)

---

## 13.1 Web2 vs Web3 Frontend

### Comparação Arquitetural

**Web2 Stack**:
```
React Frontend
    ↓ HTTP (fetch/axios)
Node.js Backend (Express)
    ↓
PostgreSQL Database
```

**Web3 Stack**:
```
React Frontend
    ↓ JSON-RPC (Ethers.js)
Ethereum Node (via Alchemy/Infura)
    ↓
Blockchain (Smart Contract State)
```

### Diferenças Fundamentais

| Aspecto | Web2 | Web3 |
|---------|------|------|
| **Auth** | JWT, OAuth, Session | Wallet signature |
| **User ID** | Email, username | Ethereum address (0x...) |
| **Persistência** | Database (MySQL, Mongo) | Blockchain state |
| **API** | REST, GraphQL | JSON-RPC (eth_call, eth_sendTransaction) |
| **Custo** | Server/cloud | Gas (usuário paga) |
| **Latência** | ~50-200ms | ~12s (block time) |
| **Reversibilidade** | ✅ UPDATE, DELETE | ❌ Imutável |

### Stack Moderno Web3

```bash
# Frontend
- React / Next.js / Vue
- Ethers.js v6 (biblioteca principal)
- Wagmi / RainbowKit (React hooks)
- TailwindCSS / Chakra UI

# Node Provider
- Alchemy (recomendado)
- Infura
- QuickNode

# Wallet
- MetaMask (mais popular)
- WalletConnect (multi-wallet)
- Coinbase Wallet
```

---

## 13.2 Ethers.js vs Web3.js

### Comparação

| Feature | Ethers.js | Web3.js |
|---------|-----------|---------|
| **Tamanho** | ~88 KB | ~200 KB |
| **Manutenção** | ✅ Ativa | ⚠️ Menos ativa |
| **TypeScript** | ✅ First-class | ⚠️ Adicionado depois |
| **Docs** | ✅ Excelente | ⚠️ Razoável |
| **Comunidade** | ✅ Grande | ✅ Maior (mais antiga) |
| **Recomendação** | ✅ Use este | Legado |

💡 **Recomendação**: **Use Ethers.js v6** (lançado em 2023, reescrito em TypeScript).

### Instalação

```bash
# Ethers.js v6
npm install ethers

# Wagmi (React hooks) - opcional mas recomendado
npm install wagmi viem

# RainbowKit (UI de wallet) - opcional
npm install @rainbow-me/rainbowkit
```

---

## 13.3 Conectar Wallet

### MetaMask Detection

```javascript
// Verificar se MetaMask está instalado
if (typeof window.ethereum !== 'undefined') {
  console.log('MetaMask installed!');
} else {
  console.log('Please install MetaMask!');
  // Redirecionar para https://metamask.io
}
```

### Conectar Wallet (Vanilla JavaScript)

```javascript
import { ethers } from 'ethers';

async function connectWallet() {
  try {
    // Request account access
    const accounts = await window.ethereum.request({
      method: 'eth_requestAccounts'
    });

    console.log('Connected:', accounts[0]);

    // Create provider
    const provider = new ethers.BrowserProvider(window.ethereum);

    // Get signer (para enviar transações)
    const signer = await provider.getSigner();

    console.log('Signer address:', await signer.getAddress());

    return { provider, signer, address: accounts[0] };
  } catch (error) {
    console.error('User rejected connection:', error);
  }
}
```

### React Component

```typescript
// WalletConnect.tsx
import { useState, useEffect } from 'react';
import { ethers } from 'ethers';

export default function WalletConnect() {
  const [address, setAddress] = useState<string | null>(null);
  const [balance, setBalance] = useState<string | null>(null);

  async function connect() {
    if (!window.ethereum) {
      alert('Please install MetaMask!');
      return;
    }

    try {
      // Request accounts
      const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
      });

      const provider = new ethers.BrowserProvider(window.ethereum);
      const signer = await provider.getSigner();
      const address = await signer.getAddress();

      // Get balance
      const balance = await provider.getBalance(address);
      const balanceInEth = ethers.formatEther(balance);

      setAddress(address);
      setBalance(balanceInEth);
    } catch (error) {
      console.error('Error connecting wallet:', error);
    }
  }

  async function disconnect() {
    setAddress(null);
    setBalance(null);
  }

  // Listen to account changes
  useEffect(() => {
    if (!window.ethereum) return;

    const handleAccountsChanged = (accounts: string[]) => {
      if (accounts.length === 0) {
        disconnect();
      } else {
        setAddress(accounts[0]);
      }
    };

    const handleChainChanged = () => {
      window.location.reload();
    };

    window.ethereum.on('accountsChanged', handleAccountsChanged);
    window.ethereum.on('chainChanged', handleChainChanged);

    return () => {
      window.ethereum.removeListener('accountsChanged', handleAccountsChanged);
      window.ethereum.removeListener('chainChanged', handleChainChanged);
    };
  }, []);

  return (
    <div>
      {!address ? (
        <button onClick={connect}>Connect Wallet</button>
      ) : (
        <div>
          <p>Address: {address.slice(0, 6)}...{address.slice(-4)}</p>
          <p>Balance: {balance} ETH</p>
          <button onClick={disconnect}>Disconnect</button>
        </div>
      )}
    </div>
  );
}
```

### Verificar Network

```javascript
async function checkNetwork() {
  const provider = new ethers.BrowserProvider(window.ethereum);
  const network = await provider.getNetwork();

  console.log('Chain ID:', network.chainId);
  console.log('Network name:', network.name);

  // Ethereum Mainnet = 1
  // Sepolia Testnet = 11155111
  // Polygon Mainnet = 137

  if (network.chainId !== 1n) {
    alert('Please switch to Ethereum Mainnet!');

    // Request network switch
    try {
      await window.ethereum.request({
        method: 'wallet_switchEthereumChain',
        params: [{ chainId: '0x1' }], // 0x1 = 1 (mainnet)
      });
    } catch (error) {
      console.error('Failed to switch network:', error);
    }
  }
}
```

---

## 13.4 Ler Dados da Blockchain

### Ler Balance

```javascript
import { ethers } from 'ethers';

// Provider (read-only, não precisa de wallet)
const provider = new ethers.JsonRpcProvider('https://eth-mainnet.g.alchemy.com/v2/YOUR_API_KEY');

// Get ETH balance
async function getBalance(address) {
  const balance = await provider.getBalance(address);
  const balanceInEth = ethers.formatEther(balance); // Wei → ETH
  console.log(`Balance: ${balanceInEth} ETH`);
  return balanceInEth;
}

// Get block number
async function getBlockNumber() {
  const blockNumber = await provider.getBlockNumber();
  console.log(`Current block: ${blockNumber}`);
  return blockNumber;
}

// Get gas price
async function getGasPrice() {
  const feeData = await provider.getFeeData();
  const gasPriceGwei = ethers.formatUnits(feeData.gasPrice, 'gwei');
  console.log(`Gas price: ${gasPriceGwei} gwei`);
  return gasPriceGwei;
}
```

### Ler Smart Contract

```javascript
// ABI do contrato (copiar do compilador ou Etherscan)
const ERC20_ABI = [
  "function name() view returns (string)",
  "function symbol() view returns (string)",
  "function balanceOf(address) view returns (uint256)",
  "function totalSupply() view returns (uint256)",
  "function decimals() view returns (uint8)"
];

// Endereço do token (USDC no Ethereum)
const USDC_ADDRESS = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';

// Create contract instance
const contract = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, provider);

// Read functions (não custa gas)
async function readToken() {
  const name = await contract.name();
  const symbol = await contract.symbol();
  const decimals = await contract.decimals();
  const totalSupply = await contract.totalSupply();

  console.log(`Name: ${name}`);
  console.log(`Symbol: ${symbol}`);
  console.log(`Decimals: ${decimals}`);
  console.log(`Total Supply: ${ethers.formatUnits(totalSupply, decimals)}`);
}

// Check user balance
async function getUserTokenBalance(userAddress) {
  const balance = await contract.balanceOf(userAddress);
  const decimals = await contract.decimals();
  const balanceFormatted = ethers.formatUnits(balance, decimals);

  console.log(`${userAddress} has ${balanceFormatted} USDC`);
  return balanceFormatted;
}
```

### React Component: Token Balance

```typescript
// TokenBalance.tsx
import { useState, useEffect } from 'react';
import { ethers } from 'ethers';

const ERC20_ABI = [
  "function balanceOf(address) view returns (uint256)",
  "function decimals() view returns (uint8)",
  "function symbol() view returns (string)"
];

interface TokenBalanceProps {
  tokenAddress: string;
  userAddress: string;
}

export default function TokenBalance({ tokenAddress, userAddress }: TokenBalanceProps) {
  const [balance, setBalance] = useState<string>('0');
  const [symbol, setSymbol] = useState<string>('');
  const [loading, setLoading] = useState<boolean>(true);

  useEffect(() => {
    async function fetchBalance() {
      try {
        const provider = new ethers.BrowserProvider(window.ethereum);
        const contract = new ethers.Contract(tokenAddress, ERC20_ABI, provider);

        const [balanceRaw, decimals, symbol] = await Promise.all([
          contract.balanceOf(userAddress),
          contract.decimals(),
          contract.symbol()
        ]);

        const balanceFormatted = ethers.formatUnits(balanceRaw, decimals);

        setBalance(balanceFormatted);
        setSymbol(symbol);
      } catch (error) {
        console.error('Error fetching balance:', error);
      } finally {
        setLoading(false);
      }
    }

    fetchBalance();
  }, [tokenAddress, userAddress]);

  if (loading) return <p>Loading...</p>;

  return (
    <div>
      <p>Balance: {balance} {symbol}</p>
    </div>
  );
}
```

---

## 13.5 Enviar Transações

### Enviar ETH

```javascript
async function sendETH(to, amountInEth) {
  const provider = new ethers.BrowserProvider(window.ethereum);
  const signer = await provider.getSigner();

  // Create transaction
  const tx = await signer.sendTransaction({
    to: to,
    value: ethers.parseEther(amountInEth) // ETH → Wei
  });

  console.log('Transaction hash:', tx.hash);

  // Wait for confirmation
  const receipt = await tx.wait();

  console.log('Transaction confirmed in block:', receipt.blockNumber);
  console.log('Gas used:', receipt.gasUsed.toString());

  return receipt;
}

// Usage
await sendETH('0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb', '0.1');
```

### Chamar Função de Smart Contract

```javascript
const ERC20_ABI = [
  "function transfer(address to, uint256 amount) returns (bool)",
  "function approve(address spender, uint256 amount) returns (bool)"
];

const USDC_ADDRESS = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';

async function transferToken(to, amount) {
  const provider = new ethers.BrowserProvider(window.ethereum);
  const signer = await provider.getSigner();

  // Contract com signer (para escrever)
  const contract = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, signer);

  // USDC tem 6 decimais
  const amountWithDecimals = ethers.parseUnits(amount, 6);

  try {
    // Enviar transação
    const tx = await contract.transfer(to, amountWithDecimals);

    console.log('Transaction sent:', tx.hash);

    // Esperar confirmação
    const receipt = await tx.wait();

    console.log('Transfer confirmed!');
    console.log('Block:', receipt.blockNumber);
    console.log('Gas used:', receipt.gasUsed.toString());

    return receipt;
  } catch (error) {
    console.error('Transfer failed:', error);
    throw error;
  }
}

// Approve (para DEX poder gastar seus tokens)
async function approveToken(spender, amount) {
  const provider = new ethers.BrowserProvider(window.ethereum);
  const signer = await provider.getSigner();
  const contract = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, signer);

  const amountWithDecimals = ethers.parseUnits(amount, 6);

  const tx = await contract.approve(spender, amountWithDecimals);
  await tx.wait();

  console.log('Approval confirmed!');
}
```

### React Component: Send Transaction

```typescript
// TransferToken.tsx
import { useState } from 'react';
import { ethers } from 'ethers';

const ERC20_ABI = ["function transfer(address to, uint256 amount) returns (bool)"];

export default function TransferToken() {
  const [recipient, setRecipient] = useState('');
  const [amount, setAmount] = useState('');
  const [loading, setLoading] = useState(false);
  const [txHash, setTxHash] = useState('');

  async function handleTransfer(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);

    try {
      const provider = new ethers.BrowserProvider(window.ethereum);
      const signer = await provider.getSigner();

      const tokenAddress = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48'; // USDC
      const contract = new ethers.Contract(tokenAddress, ERC20_ABI, signer);

      const amountWithDecimals = ethers.parseUnits(amount, 6); // USDC = 6 decimals

      const tx = await contract.transfer(recipient, amountWithDecimals);
      setTxHash(tx.hash);

      await tx.wait();
      alert('Transfer successful!');
    } catch (error: any) {
      console.error('Transfer failed:', error);
      alert(error.message);
    } finally {
      setLoading(false);
    }
  }

  return (
    <form onSubmit={handleTransfer}>
      <input
        type="text"
        placeholder="Recipient address"
        value={recipient}
        onChange={(e) => setRecipient(e.target.value)}
      />
      <input
        type="number"
        placeholder="Amount"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
      />
      <button type="submit" disabled={loading}>
        {loading ? 'Sending...' : 'Transfer'}
      </button>

      {txHash && (
        <p>
          Transaction:{' '}
          <a href={`https://etherscan.io/tx/${txHash}`} target="_blank">
            {txHash.slice(0, 10)}...
          </a>
        </p>
      )}
    </form>
  );
}
```

---

## 13.6 Escutar Eventos

### Event Listeners

```javascript
const ERC20_ABI = [
  "event Transfer(address indexed from, address indexed to, uint256 value)"
];

const USDC_ADDRESS = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';

async function listenToTransfers() {
  const provider = new ethers.JsonRpcProvider('YOUR_RPC_URL');
  const contract = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, provider);

  // Listen to all transfers
  contract.on('Transfer', (from, to, amount, event) => {
    console.log(`Transfer detected:`);
    console.log(`From: ${from}`);
    console.log(`To: ${to}`);
    console.log(`Amount: ${ethers.formatUnits(amount, 6)} USDC`);
    console.log(`Block: ${event.log.blockNumber}`);
  });

  // Listen to transfers TO specific address
  const userAddress = '0x...';
  const filter = contract.filters.Transfer(null, userAddress);

  contract.on(filter, (from, to, amount) => {
    console.log(`You received ${ethers.formatUnits(amount, 6)} USDC from ${from}`);
  });
}

// Query past events
async function getPastTransfers(userAddress) {
  const provider = new ethers.JsonRpcProvider('YOUR_RPC_URL');
  const contract = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, provider);

  // Get events from last 1000 blocks
  const currentBlock = await provider.getBlockNumber();
  const fromBlock = currentBlock - 1000;

  const filter = contract.filters.Transfer(null, userAddress);
  const events = await contract.queryFilter(filter, fromBlock, currentBlock);

  console.log(`Found ${events.length} transfers`);

  events.forEach((event) => {
    console.log(`From: ${event.args.from}`);
    console.log(`Amount: ${ethers.formatUnits(event.args.value, 6)} USDC`);
    console.log(`Block: ${event.blockNumber}`);
  });

  return events;
}
```

### React: Real-time Balance Updates

```typescript
// RealTimeBalance.tsx
import { useState, useEffect } from 'react';
import { ethers } from 'ethers';

const ERC20_ABI = [
  "function balanceOf(address) view returns (uint256)",
  "function decimals() view returns (uint8)",
  "event Transfer(address indexed from, address indexed to, uint256 value)"
];

export default function RealTimeBalance({ tokenAddress, userAddress }: any) {
  const [balance, setBalance] = useState<string>('0');

  useEffect(() => {
    const provider = new ethers.BrowserProvider(window.ethereum);
    let contract: ethers.Contract;

    async function setup() {
      contract = new ethers.Contract(tokenAddress, ERC20_ABI, provider);

      // Initial balance
      await updateBalance();

      // Listen to transfers involving this user
      const filterTo = contract.filters.Transfer(null, userAddress);
      const filterFrom = contract.filters.Transfer(userAddress, null);

      contract.on(filterTo, updateBalance);
      contract.on(filterFrom, updateBalance);
    }

    async function updateBalance() {
      const balanceRaw = await contract.balanceOf(userAddress);
      const decimals = await contract.decimals();
      const balanceFormatted = ethers.formatUnits(balanceRaw, decimals);
      setBalance(balanceFormatted);
    }

    setup();

    return () => {
      if (contract) {
        contract.removeAllListeners();
      }
    };
  }, [tokenAddress, userAddress]);

  return <div>Balance: {balance}</div>;
}
```

---

## 13.7 React Hooks para Web3

### Custom Hook: useWallet

```typescript
// hooks/useWallet.ts
import { useState, useEffect } from 'react';
import { ethers } from 'ethers';

export function useWallet() {
  const [address, setAddress] = useState<string | null>(null);
  const [provider, setProvider] = useState<ethers.BrowserProvider | null>(null);
  const [signer, setSigner] = useState<ethers.Signer | null>(null);
  const [chainId, setChainId] = useState<number | null>(null);

  async function connect() {
    if (!window.ethereum) {
      alert('Please install MetaMask!');
      return;
    }

    const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });
    const provider = new ethers.BrowserProvider(window.ethereum);
    const signer = await provider.getSigner();
    const network = await provider.getNetwork();

    setAddress(accounts[0]);
    setProvider(provider);
    setSigner(signer);
    setChainId(Number(network.chainId));
  }

  function disconnect() {
    setAddress(null);
    setProvider(null);
    setSigner(null);
    setChainId(null);
  }

  useEffect(() => {
    if (!window.ethereum) return;

    window.ethereum.on('accountsChanged', (accounts: string[]) => {
      if (accounts.length === 0) {
        disconnect();
      } else {
        setAddress(accounts[0]);
      }
    });

    window.ethereum.on('chainChanged', () => {
      window.location.reload();
    });

    return () => {
      window.ethereum.removeAllListeners();
    };
  }, []);

  return { address, provider, signer, chainId, connect, disconnect };
}
```

### Custom Hook: useContract

```typescript
// hooks/useContract.ts
import { useMemo } from 'react';
import { ethers } from 'ethers';

export function useContract(
  address: string,
  abi: any[],
  provider?: ethers.Provider | ethers.Signer
) {
  return useMemo(() => {
    if (!address || !abi || !provider) return null;
    return new ethers.Contract(address, abi, provider);
  }, [address, abi, provider]);
}
```

### Custom Hook: useTokenBalance

```typescript
// hooks/useTokenBalance.ts
import { useState, useEffect } from 'react';
import { ethers } from 'ethers';
import { useContract } from './useContract';

const ERC20_ABI = [
  "function balanceOf(address) view returns (uint256)",
  "function decimals() view returns (uint8)",
  "event Transfer(address indexed from, address indexed to, uint256 value)"
];

export function useTokenBalance(tokenAddress: string, userAddress: string, provider: any) {
  const [balance, setBalance] = useState<string>('0');
  const [loading, setLoading] = useState<boolean>(true);

  const contract = useContract(tokenAddress, ERC20_ABI, provider);

  useEffect(() => {
    if (!contract || !userAddress) return;

    async function fetchBalance() {
      try {
        const balanceRaw = await contract.balanceOf(userAddress);
        const decimals = await contract.decimals();
        const balanceFormatted = ethers.formatUnits(balanceRaw, decimals);
        setBalance(balanceFormatted);
      } catch (error) {
        console.error('Error fetching balance:', error);
      } finally {
        setLoading(false);
      }
    }

    fetchBalance();

    // Listen to transfers
    const filterTo = contract.filters.Transfer(null, userAddress);
    const filterFrom = contract.filters.Transfer(userAddress, null);

    contract.on(filterTo, fetchBalance);
    contract.on(filterFrom, fetchBalance);

    return () => {
      contract.removeAllListeners();
    };
  }, [contract, userAddress]);

  return { balance, loading };
}
```

### Usage in Component

```typescript
// App.tsx
import { useWallet } from './hooks/useWallet';
import { useTokenBalance } from './hooks/useTokenBalance';

export default function App() {
  const { address, provider, connect, disconnect } = useWallet();

  const USDC_ADDRESS = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';
  const { balance, loading } = useTokenBalance(USDC_ADDRESS, address || '', provider);

  return (
    <div>
      {!address ? (
        <button onClick={connect}>Connect Wallet</button>
      ) : (
        <div>
          <p>Address: {address.slice(0, 6)}...{address.slice(-4)}</p>
          {loading ? (
            <p>Loading balance...</p>
          ) : (
            <p>USDC Balance: {balance}</p>
          )}
          <button onClick={disconnect}>Disconnect</button>
        </div>
      )}
    </div>
  );
}
```

---

## 13.8 Error Handling

### Tipos Comuns de Erros

```typescript
async function handleTransaction() {
  try {
    const tx = await contract.transfer(to, amount);
    await tx.wait();
  } catch (error: any) {
    // User rejected
    if (error.code === 'ACTION_REJECTED') {
      alert('Transaction rejected by user');
      return;
    }

    // Insufficient funds
    if (error.code === 'INSUFFICIENT_FUNDS') {
      alert('Insufficient funds for gas');
      return;
    }

    // Contract revert
    if (error.code === 'CALL_EXCEPTION') {
      // Parse revert reason
      const reason = error.reason || error.message;
      alert(`Transaction failed: ${reason}`);
      return;
    }

    // Network error
    if (error.code === 'NETWORK_ERROR') {
      alert('Network error. Please try again.');
      return;
    }

    // Generic error
    console.error('Unexpected error:', error);
    alert('Transaction failed');
  }
}
```

### Parse Revert Reason

```typescript
async function getRevertReason(txHash: string, provider: ethers.Provider) {
  const tx = await provider.getTransaction(txHash);
  if (!tx) throw new Error('Transaction not found');

  try {
    await provider.call(tx, tx.blockNumber);
  } catch (error: any) {
    // Error data contém revert reason
    const reason = error.reason || 'Unknown error';
    return reason;
  }
}
```

### User-Friendly Error Messages

```typescript
function getErrorMessage(error: any): string {
  const errorMessages: Record<string, string> = {
    'ACTION_REJECTED': 'You rejected the transaction',
    'INSUFFICIENT_FUNDS': 'Insufficient funds for gas',
    'UNPREDICTABLE_GAS_LIMIT': 'Cannot estimate gas. Transaction may fail.',
    'NONCE_EXPIRED': 'Transaction nonce expired. Please try again.',
    'REPLACEMENT_UNDERPRICED': 'Gas price too low. Increase gas price.',
  };

  return errorMessages[error.code] || 'Transaction failed. Please try again.';
}
```

---

## 13.9 Boas Práticas de UX

### 1. Loading States

```typescript
function TransferButton() {
  const [status, setStatus] = useState<'idle' | 'pending' | 'confirming' | 'success'>('idle');

  async function handleTransfer() {
    setStatus('pending'); // User aprovou no MetaMask

    const tx = await contract.transfer(to, amount);

    setStatus('confirming'); // Aguardando confirmação

    await tx.wait();

    setStatus('success');

    setTimeout(() => setStatus('idle'), 3000);
  }

  return (
    <button disabled={status !== 'idle'}>
      {status === 'idle' && 'Transfer'}
      {status === 'pending' && 'Check your wallet...'}
      {status === 'confirming' && 'Confirming...'}
      {status === 'success' && 'Success!'}
    </button>
  );
}
```

### 2. Transaction Links

```typescript
function TxLink({ txHash, chainId }: { txHash: string; chainId: number }) {
  const explorers: Record<number, string> = {
    1: 'https://etherscan.io',
    11155111: 'https://sepolia.etherscan.io',
    137: 'https://polygonscan.com',
  };

  const explorer = explorers[chainId] || 'https://etherscan.io';

  return (
    <a href={`${explorer}/tx/${txHash}`} target="_blank" rel="noopener noreferrer">
      View on Explorer ↗
    </a>
  );
}
```

### 3. Gas Estimação

```typescript
async function estimateGas() {
  const contract = new ethers.Contract(address, abi, signer);

  try {
    // Estimate gas
    const gasEstimate = await contract.transfer.estimateGas(to, amount);

    // Get gas price
    const feeData = await provider.getFeeData();

    // Calculate total cost
    const gasCost = gasEstimate * feeData.gasPrice!;
    const gasCostInEth = ethers.formatEther(gasCost);

    console.log(`Estimated gas: ${gasEstimate.toString()}`);
    console.log(`Gas cost: ${gasCostInEth} ETH`);

    // Show to user BEFORE they confirm
    const confirmMessage = `This will cost approximately ${gasCostInEth} ETH in gas. Continue?`;

    if (confirm(confirmMessage)) {
      const tx = await contract.transfer(to, amount);
      await tx.wait();
    }
  } catch (error) {
    console.error('Gas estimation failed. Transaction will likely fail.');
  }
}
```

### 4. Network Switch Prompt

```typescript
async function switchToNetwork(chainId: number) {
  const networks: Record<number, { chainId: string; chainName: string; rpcUrls: string[] }> = {
    1: {
      chainId: '0x1',
      chainName: 'Ethereum Mainnet',
      rpcUrls: ['https://mainnet.infura.io/v3/YOUR_KEY']
    },
    137: {
      chainId: '0x89',
      chainName: 'Polygon Mainnet',
      rpcUrls: ['https://polygon-rpc.com']
    }
  };

  const network = networks[chainId];

  try {
    await window.ethereum.request({
      method: 'wallet_switchEthereumChain',
      params: [{ chainId: network.chainId }],
    });
  } catch (switchError: any) {
    // Chain not added to MetaMask
    if (switchError.code === 4902) {
      try {
        await window.ethereum.request({
          method: 'wallet_addEthereumChain',
          params: [network],
        });
      } catch (addError) {
        console.error('Failed to add network:', addError);
      }
    }
  }
}
```

### 5. Optimistic UI Updates

```typescript
function OptimisticTransfer() {
  const [balance, setBalance] = useState('1000');

  async function transfer(amount: string) {
    // 1. Update UI imediatamente (optimistic)
    const newBalance = (parseFloat(balance) - parseFloat(amount)).toString();
    setBalance(newBalance);

    try {
      // 2. Enviar transação
      const tx = await contract.transfer(to, ethers.parseEther(amount));
      await tx.wait();

      // 3. Sucesso - UI já estava certo
    } catch (error) {
      // 4. Erro - reverter UI
      setBalance(balance); // Restore old balance
      alert('Transfer failed');
    }
  }

  return <div>Balance: {balance} ETH</div>;
}
```

---

## 📖 Glossário

**Provider**
> Conexão read-only com blockchain (pode ler, não pode escrever).
> **Analogia**: Como API key read-only.

**Signer**
> Provider + private key (pode escrever na blockchain).
> **Uso**: Necessário para enviar transações.

**ABI (Application Binary Interface)**
> JSON que descreve funções e eventos do contrato.
> **Analogia**: Como schema de API REST (OpenAPI/Swagger).

**Wei**
> Menor unidade de Ether (1 ETH = 10^18 Wei).
> **Conversão**: Use `parseEther()` e `formatEther()`.

**Gas Estimation**
> Calcular quanto gas uma transação vai custar antes de enviar.
> **Por que**: Prevenir erros de "out of gas".

**Event Listener**
> Escutar eventos emitidos por smart contracts.
> **Uso**: Updates em tempo real (ex: novo transfer).

**Transaction Receipt**
> Resultado de transação confirmada (block number, gas used, logs).

**MetaMask**
> Wallet browser mais popular.
> **Injecta**: `window.ethereum` object.

---

## 🔒 Security Checklist: Frontend

### Wallet Connection
- [ ] Verificar se MetaMask está instalado
- [ ] Nunca pedir private key (MetaMask gerencia)
- [ ] Verificar network/chainId
- [ ] Lidar com account/network changes

### Transactions
- [ ] Validar inputs antes de enviar
- [ ] Estimar gas e mostrar custo ao user
- [ ] Permitir ajuste de gas price
- [ ] Mostrar transaction hash e link para explorer
- [ ] Error handling robusto

### Data Display
- [ ] Formatar addresses (mostrar 0x1234...5678)
- [ ] Formatar números (usar formatEther, formatUnits)
- [ ] Verificar decimals de token
- [ ] Atualizar UI quando transação confirmar

### General
- [ ] NUNCA armazenar private keys
- [ ] NUNCA fazer requests HTTP para sites suspeitos
- [ ] Verificar ABI e address do contrato
- [ ] Rate limiting em RPC providers

---

## 📝 Exercícios Práticos

### Exercício 1: Token Dashboard

**Objetivo**: Criar dashboard de ERC-20 tokens.

**Features**:
- Conectar wallet
- Mostrar lista de tokens (USDC, DAI, LINK)
- Balance de cada token
- Botão para transferir
- Histórico de transfers (eventos)

**Tech**: React + Ethers.js + TailwindCSS

---

### Exercício 2: DEX Interface

**Objetivo**: UI para DEX do Cap 10.

**Features**:
- Swap tokens
- Add/Remove liquidez
- Ver LP token balance
- Real-time price updates

<details>
<summary>💡 Dica</summary>

Use hooks customizados:
- `useContract` para instanciar DEX
- `useTokenBalance` para cada token
- `useSwap` para lógica de swap
</details>

---

### Exercício 3: NFT Gallery

**Objetivo**: Galeria de NFTs do usuário.

**Features**:
- Conectar wallet
- Fetch NFTs (via Alchemy NFT API ou events)
- Mostrar metadata (IPFS)
- Transfer NFT para outro endereço

---

## 📚 Recursos Adicionais

### Documentação

1. **[Ethers.js Docs](https://docs.ethers.org/v6/)**
   - Referência completa v6

2. **[Wagmi](https://wagmi.sh/)**
   - React hooks para Ethereum

3. **[RainbowKit](https://www.rainbowkit.com/)**
   - UI de wallet connection

### Ferramentas

- **[Alchemy](https://www.alchemy.com/)** - Node provider + APIs
- **[useDApp](https://usedapp.io/)** - React hooks
- **[Web3Modal](https://web3modal.com/)** - Multi-wallet modal

---

## 🎯 Próximos Passos

**Você agora sabe**:
- ✅ Conectar wallet (MetaMask)
- ✅ Ler dados on-chain (balances, contracts)
- ✅ Enviar transações
- ✅ Escutar eventos em tempo real
- ✅ React hooks customizados
- ✅ Error handling
- ✅ UX best practices

**Próximo Capítulo**: [Capítulo 14 - Indexing](./EBOOK_CAPITULO_14_INDEXING.md)
- The Graph (subgraphs)
- Por que indexing é necessário
- GraphQL queries
- Alternativas (Moralis, Alchemy)

**Aplicação Full-Stack**:
- Cap 10 (DEX) + Cap 11 (Oracles) + Cap 13 (Frontend) = DEX completo com UI
- Cap 9 (NFTs) + Cap 13 = NFT marketplace
- Cap 6 (Staking) + Cap 13 = Staking dashboard

💡 **Projeto Sugerido**: Clone de Uniswap interface + seu DEX do Cap 10.

---

**Autor**: Baseado em roadmap ITA Blockchain Club + Ethers.js docs
**Última Atualização**: 2025-11-14
