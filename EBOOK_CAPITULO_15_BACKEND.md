# Capítulo 15: Backend - APIs, IPFS, Arquitetura Híbrida

> **Para Desenvolvedores Experientes**: Nem tudo vai on-chain. Armazenar 1MB on-chain custa ~$100,000 em gas. Computação complexa? Impossível (gas limits). Backend continua existindo em Web3 - mas agora para computação off-chain, IPFS pinning, webhooks de eventos, e orquestração. É arquitetura híbrida: on-chain para trust, off-chain para eficiência.

---

## Índice
- [15.1 Quando Você Precisa de Backend](#151-quando-você-precisa-de-backend)
- [15.2 Node.js + Ethers.js](#152-nodejs--ethersjs)
- [15.3 Webhooks para Eventos](#153-webhooks-para-eventos)
- [15.4 IPFS Pinning](#154-ipfs-pinning)
- [15.5 Database Off-Chain](#155-database-off-chain)
- [15.6 Computação Off-Chain, Verificação On-Chain](#156-computação-off-chain-verificação-on-chain)
- [15.7 Arquitetura Híbrida](#157-arquitetura-híbrida)

---

## 15.1 Quando Você Precisa de Backend

### On-Chain vs Off-Chain

| Dado/Lógica | On-Chain | Off-Chain (Backend) |
|-------------|----------|---------------------|
| **Ownership** (quem é dono de NFT) | ✅ | ❌ |
| **Balances** (quanto cada um tem) | ✅ | ❌ |
| **Business logic** crítica (transferências) | ✅ | ❌ |
| **Metadata** (imagens, descrições) | ❌ ($$$) | ✅ (IPFS) |
| **Analytics** (dashboards) | ❌ (lento) | ✅ |
| **Computação pesada** (IA, rendering) | ❌ (gas) | ✅ |
| **Notificações** (email, push) | ❌ | ✅ |
| **Caching** | ❌ | ✅ |

### Casos de Uso de Backend

✅ **Use backend para**:
1. **IPFS Pinning**: Upload e pin de arquivos
2. **Webhooks**: Escutar eventos e notificar users
3. **Caching**: Guardar dados para queries rápidas
4. **Computação complexa**: IA, machine learning, rendering
5. **APIs**: Agregação de dados de múltiplas fontes
6. **Email/SMS**: Notificações off-chain
7. **Payment rails**: Integrar com Stripe, fiat

❌ **NÃO use backend para**:
- Armazenar ownership (use blockchain)
- Lógica de negócio crítica (use smart contract)
- Anything que precisa ser trustless

---

## 15.2 Node.js + Ethers.js

### Setup Básico

```bash
# Create project
mkdir nft-backend
cd nft-backend
npm init -y

# Install dependencies
npm install express ethers dotenv
npm install --save-dev typescript @types/node @types/express
```

### Server Básico

```typescript
// server.ts
import express from 'express';
import { ethers } from 'ethers';
import dotenv from 'dotenv';

dotenv.config();

const app = express();
app.use(express.json());

// Setup provider
const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);

// Contract instance
const NFT_ADDRESS = process.env.NFT_CONTRACT_ADDRESS!;
const NFT_ABI = [
  "function ownerOf(uint256 tokenId) view returns (address)",
  "function tokenURI(uint256 tokenId) view returns (string)"
];
const nftContract = new ethers.Contract(NFT_ADDRESS, NFT_ABI, provider);

// Routes
app.get('/nft/:tokenId', async (req, res) => {
  try {
    const tokenId = req.params.tokenId;

    // Get owner from blockchain
    const owner = await nftContract.ownerOf(tokenId);

    // Get metadata URI
    const tokenURI = await nftContract.tokenURI(tokenId);

    res.json({
      tokenId,
      owner,
      tokenURI
    });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/balance/:address', async (req, res) => {
  try {
    const balance = await provider.getBalance(req.params.address);
    res.json({
      address: req.params.address,
      balance: ethers.formatEther(balance)
    });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Enviar Transações do Backend

⚠️ **Security**: Guardar private key segura (KMS, Vault, etc.)!

```typescript
// wallet.ts
import { ethers } from 'ethers';

const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);

// ⚠️ NUNCA commitar private key!
const wallet = new ethers.Wallet(process.env.PRIVATE_KEY!, provider);

// Mint NFT para user (gasless para user)
app.post('/mint', async (req, res) => {
  try {
    const { to } = req.body;

    // Validar address
    if (!ethers.isAddress(to)) {
      return res.status(400).json({ error: 'Invalid address' });
    }

    // Backend paga o gas
    const nftContract = new ethers.Contract(
      NFT_ADDRESS,
      ["function mint(address to) returns (uint256)"],
      wallet // Signer, não provider
    );

    const tx = await nftContract.mint(to);
    const receipt = await tx.wait();

    res.json({
      success: true,
      transactionHash: receipt.hash,
      blockNumber: receipt.blockNumber
    });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});
```

### Rate Limiting

```typescript
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutos
  max: 100, // Max 100 requests por IP
  message: 'Too many requests, please try again later.'
});

app.use('/api/', limiter);
```

---

## 15.3 Webhooks para Eventos

### Escutar Eventos e Notificar

```typescript
// eventListener.ts
import { ethers } from 'ethers';
import nodemailer from 'nodemailer';

const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);

const ERC721_ABI = [
  "event Transfer(address indexed from, address indexed to, uint256 indexed tokenId)"
];

const nftContract = new ethers.Contract(NFT_ADDRESS, ERC721_ABI, provider);

// Email transporter
const transporter = nodemailer.createTransport({
  service: 'gmail',
  auth: {
    user: process.env.EMAIL_USER,
    pass: process.env.EMAIL_PASS
  }
});

// Listen to Transfer events
nftContract.on('Transfer', async (from, to, tokenId, event) => {
  console.log(`NFT #${tokenId} transferred from ${from} to ${to}`);

  // Fetch user email from database (exemplo)
  const userEmail = await getUserEmail(to);

  if (userEmail) {
    // Send email notification
    await transporter.sendMail({
      from: 'nft@example.com',
      to: userEmail,
      subject: `You received NFT #${tokenId}!`,
      html: `
        <h1>Congratulations!</h1>
        <p>You received NFT #${tokenId}</p>
        <p>View on OpenSea: https://opensea.io/assets/${NFT_ADDRESS}/${tokenId}</p>
      `
    });

    console.log(`Email sent to ${userEmail}`);
  }

  // Store in database
  await storeTransfer({
    from,
    to,
    tokenId: tokenId.toString(),
    blockNumber: event.blockNumber,
    transactionHash: event.transactionHash
  });
});

async function getUserEmail(address: string): Promise<string | null> {
  // Query database
  // return email or null
  return null;
}

async function storeTransfer(transfer: any): Promise<void> {
  // Save to database
}

console.log('Listening to NFT transfers...');
```

### Webhook Endpoints

```typescript
// webhooks.ts
import crypto from 'crypto';

// Alchemy Webhook (exemplo)
app.post('/webhook/alchemy', (req, res) => {
  // Verificar signature
  const signature = req.headers['x-alchemy-signature'] as string;
  const payload = JSON.stringify(req.body);

  const expectedSignature = crypto
    .createHmac('sha256', process.env.ALCHEMY_WEBHOOK_SECRET!)
    .update(payload)
    .digest('hex');

  if (signature !== expectedSignature) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  // Process webhook
  const { event } = req.body;

  if (event.activity) {
    event.activity.forEach((activity: any) => {
      console.log('Activity:', activity);
      // Handle NFT transfer, mint, etc.
    });
  }

  res.status(200).json({ success: true });
});
```

---

## 15.4 IPFS Pinning

### O Que É IPFS

**IPFS = InterPlanetary File System**

- Distributed storage (não centralizado)
- Content-addressed (CID = hash do conteúdo)
- Imutável (mesmo conteúdo = mesmo CID)

```
Centralized (AWS S3):
https://mybucket.s3.amazonaws.com/image.png
→ Single point of failure

Decentralized (IPFS):
ipfs://QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG
→ Qualquer IPFS node pode servir
```

### Upload para IPFS

```typescript
// ipfs.ts
import { create } from 'ipfs-http-client';
import fs from 'fs';

// Conectar a IPFS node (local ou Infura/Pinata)
const ipfs = create({
  host: 'ipfs.infura.io',
  port: 5001,
  protocol: 'https',
  headers: {
    authorization: 'Basic ' + Buffer.from(
      process.env.INFURA_PROJECT_ID + ':' + process.env.INFURA_PROJECT_SECRET
    ).toString('base64')
  }
});

// Upload file
async function uploadFile(filePath: string) {
  const file = fs.readFileSync(filePath);

  const result = await ipfs.add(file);

  console.log('IPFS CID:', result.cid.toString());
  console.log('IPFS URL:', `ipfs://${result.cid}`);
  console.log('Gateway URL:', `https://ipfs.io/ipfs/${result.cid}`);

  return result.cid.toString();
}

// Upload JSON metadata
async function uploadMetadata(metadata: any) {
  const result = await ipfs.add(JSON.stringify(metadata));
  return result.cid.toString();
}

// Example: NFT metadata
async function createNFTMetadata(name: string, description: string, imageCID: string) {
  const metadata = {
    name,
    description,
    image: `ipfs://${imageCID}`,
    attributes: [
      { trait_type: 'Color', value: 'Blue' },
      { trait_type: 'Rarity', value: 'Legendary' }
    ]
  };

  const metadataCID = await uploadMetadata(metadata);
  return metadataCID;
}

// Usage
const imageCID = await uploadFile('./nft-image.png');
const metadataCID = await createNFTMetadata('My NFT', 'Cool NFT', imageCID);

console.log('Metadata URI:', `ipfs://${metadataCID}`);
```

### Pinning Service (Pinata)

```typescript
import axios from 'axios';
import FormData from 'form-data';
import fs from 'fs';

const PINATA_API_KEY = process.env.PINATA_API_KEY!;
const PINATA_SECRET_KEY = process.env.PINATA_SECRET_KEY!;

async function pinFileToPinata(filePath: string) {
  const url = 'https://api.pinata.cloud/pinning/pinFileToIPFS';

  const data = new FormData();
  data.append('file', fs.createReadStream(filePath));

  const metadata = JSON.stringify({
    name: 'NFT Image',
  });
  data.append('pinataMetadata', metadata);

  const response = await axios.post(url, data, {
    maxBodyLength: Infinity,
    headers: {
      'Content-Type': `multipart/form-data; boundary=${data.getBoundary()}`,
      'pinata_api_key': PINATA_API_KEY,
      'pinata_secret_api_key': PINATA_SECRET_KEY
    }
  });

  return response.data.IpfsHash;
}

async function pinJSONToPinata(json: any) {
  const url = 'https://api.pinata.cloud/pinning/pinJSONToIPFS';

  const response = await axios.post(url, json, {
    headers: {
      'pinata_api_key': PINATA_API_KEY,
      'pinata_secret_api_key': PINATA_SECRET_KEY
    }
  });

  return response.data.IpfsHash;
}
```

### API Endpoint: Upload NFT

```typescript
import multer from 'multer';

const upload = multer({ dest: 'uploads/' });

app.post('/upload-nft', upload.single('image'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file uploaded' });
    }

    const { name, description } = req.body;

    // 1. Upload image to IPFS
    const imageCID = await pinFileToPinata(req.file.path);

    // 2. Create metadata
    const metadata = {
      name,
      description,
      image: `ipfs://${imageCID}`,
      attributes: []
    };

    // 3. Upload metadata to IPFS
    const metadataCID = await pinJSONToPinata(metadata);

    // 4. Delete temp file
    fs.unlinkSync(req.file.path);

    res.json({
      imageCID,
      metadataCID,
      metadataURI: `ipfs://${metadataCID}`
    });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});
```

---

## 15.5 Database Off-Chain

### Quando Usar Database

✅ **Use para**:
- User profiles (email, preferences)
- Cache de dados blockchain
- Analytics
- Session data
- Dados não-críticos

❌ **Não use para**:
- Ownership (use blockchain)
- Balances (use blockchain)
- Anything que precisa ser trustless

### Schema Example (PostgreSQL)

```sql
-- Users table
CREATE TABLE users (
  address VARCHAR(42) PRIMARY KEY,
  email VARCHAR(255) UNIQUE,
  username VARCHAR(50),
  avatar_url TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- NFT cache (para queries rápidas)
CREATE TABLE nfts (
  contract_address VARCHAR(42),
  token_id NUMERIC,
  owner VARCHAR(42),
  token_uri TEXT,
  metadata JSONB,
  last_updated TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (contract_address, token_id)
);

-- Transfer events (indexado)
CREATE TABLE transfers (
  id SERIAL PRIMARY KEY,
  contract_address VARCHAR(42),
  from_address VARCHAR(42),
  to_address VARCHAR(42),
  token_id NUMERIC,
  block_number INTEGER,
  transaction_hash VARCHAR(66),
  timestamp INTEGER,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_transfers_from ON transfers(from_address);
CREATE INDEX idx_transfers_to ON transfers(to_address);
CREATE INDEX idx_transfers_contract ON transfers(contract_address);
```

### API com Database

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL
});

// Register user
app.post('/users/register', async (req, res) => {
  const { address, email, username } = req.body;

  try {
    const result = await pool.query(
      'INSERT INTO users (address, email, username) VALUES ($1, $2, $3) RETURNING *',
      [address, email, username]
    );

    res.json(result.rows[0]);
  } catch (error: any) {
    if (error.code === '23505') { // Unique violation
      res.status(400).json({ error: 'User already exists' });
    } else {
      res.status(500).json({ error: error.message });
    }
  }
});

// Get user profile
app.get('/users/:address', async (req, res) => {
  const { address } = req.params;

  const result = await pool.query(
    'SELECT * FROM users WHERE address = $1',
    [address]
  );

  if (result.rows.length === 0) {
    return res.status(404).json({ error: 'User not found' });
  }

  res.json(result.rows[0]);
});

// Get user's NFTs (cached)
app.get('/users/:address/nfts', async (req, res) => {
  const { address } = req.params;

  const result = await pool.query(
    'SELECT * FROM nfts WHERE owner = $1',
    [address]
  );

  res.json(result.rows);
});
```

---

## 15.6 Computação Off-Chain, Verificação On-Chain

### Pattern: Compute Off-Chain, Verify On-Chain

**Problema**: Computação complexa é cara on-chain (gas).

**Solução**: Compute no backend, envie resultado + prova on-chain.

### Example: Whitelist Merkle Tree

```typescript
// generateMerkleTree.ts (Backend)
import { StandardMerkleTree } from "@openzeppelin/merkle-tree";
import fs from 'fs';

// Whitelist de addresses
const whitelist = [
  "0x1111111111111111111111111111111111111111",
  "0x2222222222222222222222222222222222222222",
  "0x3333333333333333333333333333333333333333",
  // ... 10,000 addresses
];

// Gerar Merkle Tree
const tree = StandardMerkleTree.of(
  whitelist.map(addr => [addr]),
  ["address"]
);

// Root (vai on-chain)
const root = tree.root;
console.log('Merkle Root:', root);

// Save tree para gerar proofs depois
fs.writeFileSync('merkle-tree.json', JSON.stringify(tree.dump()));

// Gerar proof para um address
function getProof(address: string) {
  for (const [i, v] of tree.entries()) {
    if (v[0] === address) {
      return tree.getProof(i);
    }
  }
  throw new Error('Address not in whitelist');
}

// API endpoint
app.get('/whitelist/proof/:address', (req, res) => {
  try {
    const proof = getProof(req.params.address);
    res.json({ proof, root });
  } catch (error: any) {
    res.status(404).json({ error: error.message });
  }
});
```

**Smart Contract**:
```solidity
// WhitelistNFT.sol
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";

contract WhitelistNFT {
    bytes32 public merkleRoot;

    constructor(bytes32 _merkleRoot) {
        merkleRoot = _merkleRoot;
    }

    function mint(bytes32[] calldata proof) external {
        // Verificar se caller está na whitelist
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender));
        require(MerkleProof.verify(proof, merkleRoot, leaf), "Not whitelisted");

        // Mint NFT...
    }
}
```

**Frontend**:
```typescript
// Fetch proof do backend
const response = await fetch(`/whitelist/proof/${userAddress}`);
const { proof, root } = await response.json();

// Mint com proof
await nftContract.mint(proof);
```

💡 **Vantagem**: Armazenar 10k addresses on-chain = $$$. Merkle root = apenas 32 bytes!

---

## 15.7 Arquitetura Híbrida

### Exemplo: NFT Marketplace

```
┌─────────────────────────────────────────────────────┐
│               FRONTEND (React)                      │
│  - Browse NFTs                                      │
│  - Connect wallet                                   │
│  - Buy/List NFTs                                    │
└───────────┬─────────────────────────┬───────────────┘
            │                         │
            ▼                         ▼
┌───────────────────────┐   ┌─────────────────────────┐
│   SMART CONTRACT      │   │     BACKEND (Node.js)   │
│   (Ethereum)          │   │                         │
│ - Ownership           │   │ - IPFS Pinning          │
│ - Transfers           │   │ - Indexing (cache)      │
│ - Escrow              │   │ - Notifications         │
└───────────────────────┘   │ - API (search, filter)  │
                            │ - Database (profiles)   │
                            └─────────────────────────┘
                                      │
                                      ▼
                            ┌─────────────────────────┐
                            │    DATABASE (Postgres)  │
                            │ - Users                 │
                            │ - NFT cache             │
                            │ - Listings              │
                            │ - Analytics             │
                            └─────────────────────────┘
```

### Fluxo: List NFT for Sale

```typescript
// 1. Frontend: Approve NFT para marketplace
await nftContract.approve(MARKETPLACE_ADDRESS, tokenId);

// 2. Frontend: Upload metadata se necessário (via backend)
const response = await fetch('/api/nfts/metadata', {
  method: 'POST',
  body: JSON.stringify({
    tokenId,
    price: ethers.parseEther('1.0'),
    description: 'Cool NFT'
  })
});

// 3. Frontend: List on-chain
await marketplaceContract.list(NFT_ADDRESS, tokenId, price);

// 4. Backend: Event listener detected listing
// → Cache in database
// → Send notification to followers
```

### Exemplo Completo: Backend + Smart Contract

**Smart Contract**:
```solidity
contract Marketplace {
    struct Listing {
        address seller;
        uint256 price;
        bool active;
    }

    mapping(address => mapping(uint256 => Listing)) public listings;

    event Listed(address indexed nftContract, uint256 indexed tokenId, address seller, uint256 price);
    event Sold(address indexed nftContract, uint256 indexed tokenId, address buyer, uint256 price);

    function list(address nftContract, uint256 tokenId, uint256 price) external {
        IERC721(nftContract).transferFrom(msg.sender, address(this), tokenId);

        listings[nftContract][tokenId] = Listing({
            seller: msg.sender,
            price: price,
            active: true
        });

        emit Listed(nftContract, tokenId, msg.sender, price);
    }

    function buy(address nftContract, uint256 tokenId) external payable {
        Listing storage listing = listings[nftContract][tokenId];

        require(listing.active, "Not listed");
        require(msg.value >= listing.price, "Insufficient payment");

        listing.active = false;

        IERC721(nftContract).transferFrom(address(this), msg.sender, tokenId);
        payable(listing.seller).transfer(listing.price);

        emit Sold(nftContract, tokenId, msg.sender, listing.price);
    }
}
```

**Backend**:
```typescript
// Listen to events and cache
marketplaceContract.on('Listed', async (nftContract, tokenId, seller, price, event) => {
  // 1. Fetch metadata
  const nftContractInstance = new ethers.Contract(nftContract, ERC721_ABI, provider);
  const tokenURI = await nftContractInstance.tokenURI(tokenId);

  // 2. Fetch from IPFS
  const metadata = await fetch(tokenURI.replace('ipfs://', 'https://ipfs.io/ipfs/'));
  const metadataJSON = await metadata.json();

  // 3. Cache in database
  await pool.query(
    `INSERT INTO listings (nft_contract, token_id, seller, price, metadata, active)
     VALUES ($1, $2, $3, $4, $5, true)
     ON CONFLICT (nft_contract, token_id) DO UPDATE
     SET seller = $3, price = $4, active = true`,
    [nftContract, tokenId.toString(), seller, price.toString(), metadataJSON]
  );

  console.log(`Cached listing: ${nftContract}/${tokenId}`);
});

// API: Search listings
app.get('/listings', async (req, res) => {
  const { search, minPrice, maxPrice, sort = 'price_asc' } = req.query;

  let query = 'SELECT * FROM listings WHERE active = true';
  const params: any[] = [];

  if (search) {
    query += ` AND metadata->>'name' ILIKE $${params.length + 1}`;
    params.push(`%${search}%`);
  }

  if (minPrice) {
    query += ` AND price >= $${params.length + 1}`;
    params.push(minPrice);
  }

  if (maxPrice) {
    query += ` AND price <= $${params.length + 1}`;
    params.push(maxPrice);
  }

  if (sort === 'price_asc') query += ' ORDER BY price ASC';
  if (sort === 'price_desc') query += ' ORDER BY price DESC';

  const result = await pool.query(query, params);
  res.json(result.rows);
});
```

---

## 📖 Glossário

**Hybrid Architecture**
> Combinar on-chain (smart contracts) com off-chain (backend, IPFS).
> **Por que**: Eficiência, custo, UX.

**IPFS**
> InterPlanetary File System - storage descentralizado e content-addressed.
> **Uso**: NFT images, metadata.

**Pinning**
> Garantir que arquivo permaneça disponível no IPFS.
> **Serviços**: Pinata, Infura, Web3.Storage.

**CID (Content Identifier)**
> Hash único de conteúdo no IPFS.
> **Exemplo**: `QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG`.

**Webhook**
> Callback HTTP quando evento acontece.
> **Uso**: Notificações, automações.

**Merkle Tree**
> Estrutura de dados para provas criptográficas eficientes.
> **Uso**: Whitelists, snapshots.

**Off-Chain Computation**
> Computação feita fora da blockchain (backend).
> **Verificação**: Proof on-chain.

---

## 🔒 Security Checklist

### Backend
- [ ] NUNCA commitar private keys
- [ ] Use environment variables (.env)
- [ ] KMS para prod (AWS KMS, Vault)
- [ ] Rate limiting em APIs
- [ ] Input validation
- [ ] CORS configurado corretamente

### IPFS
- [ ] Pin arquivos importantes (não só upload)
- [ ] Validar conteúdo antes de pin
- [ ] Backup de CIDs críticos
- [ ] Use gateway confiável (Pinata, Infura)

### Database
- [ ] Parametrized queries (evitar SQL injection)
- [ ] Indexes em colunas filtradas
- [ ] Backup regular
- [ ] Não armazenar dados críticos (ownership → blockchain)

### Webhooks
- [ ] Verificar signatures
- [ ] Idempotency (não processar evento duas vezes)
- [ ] Timeout handling
- [ ] Retry logic

---

## 📝 Exercícios

### Exercício 1: NFT Metadata API

**Objetivo**: Backend para upload de NFT metadata.

**Features**:
- Upload image → IPFS
- Generate metadata JSON → IPFS
- Return metadata URI
- Cache em database

---

### Exercício 2: Event Notification Service

**Objetivo**: Notificar users sobre eventos.

**Features**:
- Listen Transfer events
- Send email quando recebe NFT
- Web push notifications
- Telegram bot

---

### Exercício 3: Gasless Transactions

**Objetivo**: Backend paga gas por users (meta-transactions).

**Flow**:
- User assina mensagem off-chain
- Backend verifica signature
- Backend envia transação on-chain
- User não paga gas

---

## 📚 Recursos

### IPFS
1. **[IPFS Docs](https://docs.ipfs.tech/)**
2. **[Pinata](https://www.pinata.cloud/)**
3. **[Web3.Storage](https://web3.storage/)**

### Backend
- **[Ethers.js Docs](https://docs.ethers.org/v6/)**
- **[Express](https://expressjs.com/)**
- **[Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)**

---

## 🎯 Próximos Passos

**Você agora sabe**:
- ✅ Quando usar backend vs on-chain
- ✅ Node.js + Ethers.js
- ✅ Webhooks e event listeners
- ✅ IPFS pinning
- ✅ Database para cache
- ✅ Arquitetura híbrida

**Próximo Capítulo**: [Capítulo 16 - DevOps](./EBOOK_CAPITULO_16_DEVOPS.md)
- CI/CD para smart contracts
- Automated testing
- Gas reporting
- Contract verification

**Stack Completo**:
- Smart Contract + Backend + Frontend + Indexing = dApp production-ready!

💡 **Projeto**: NFT marketplace completo com backend, IPFS, database, notifications.

---

**Autor**: Baseado em roadmap ITA Blockchain Club + práticas da indústria
**Última Atualização**: 2025-11-14
