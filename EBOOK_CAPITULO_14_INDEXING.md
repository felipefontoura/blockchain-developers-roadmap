# Capítulo 14: Indexing - The Graph e Event Listeners

> **Para Desenvolvedores Experientes**: Se você já trabalhou com databases, sabe que `SELECT * FROM users WHERE created_at > '2024-01-01'` é trivial. Em blockchain, essa query simples pode custar milhares de RPC calls e minutos de processamento. The Graph é basicamente um "database index" para blockchain - transforma eventos em queryable GraphQL API.

---

## Índice
- [14.1 O Problema de Querying](#141-o-problema-de-querying)
- [14.2 The Graph - Subgraphs](#142-the-graph---subgraphs)
- [14.3 Criando um Subgraph](#143-criando-um-subgraph)
- [14.4 GraphQL Queries](#144-graphql-queries)
- [14.5 Event Listeners Customizados](#145-event-listeners-customizados)
- [14.6 Alternativas ao The Graph](#146-alternativas-ao-the-graph)
- [14.7 Quando Usar Cada Solução](#147-quando-usar-cada-solução)

---

## 14.1 O Problema de Querying

### Comparação: SQL vs Blockchain

**Web2 (SQL)**:
```sql
-- Buscar todos transfers de Alice nos últimos 30 dias
SELECT * FROM transfers
WHERE from_address = '0xAlice'
  AND timestamp > NOW() - INTERVAL 30 DAY
ORDER BY timestamp DESC
LIMIT 100;

-- Execução: ~10ms
```

**Web3 (Blockchain sem indexing)**:
```javascript
// ❌ PÉSSIMO: Query todos blocos dos últimos 30 dias
const currentBlock = await provider.getBlockNumber();
const blocksPerDay = 7200; // ~12s por bloco
const fromBlock = currentBlock - (30 * blocksPerDay);

const filter = contract.filters.Transfer('0xAlice', null);
const events = await contract.queryFilter(filter, fromBlock, currentBlock);

// Execução: 5-30 minutos! 🐌
// Cost: 200,000+ RPC calls
// Pode exceder rate limits
```

### Por Que É Difícil?

**Limitações da Blockchain**:
1. ❌ Não há SQL - apenas logs/events
2. ❌ Não há indexes - scan sequencial
3. ❌ Não há agregações - tudo client-side
4. ❌ Não há JOINs - múltiplas queries
5. ❌ Rate limits em RPC providers

**Exemplo Real**:
```javascript
// Buscar total de transações de token
// Sem indexing: Scan TODOS os blocos desde deploy
// USDC deploy (block 6_082_465) até agora (~19M blocos)
// = Impossível sem indexing!
```

### Soluções

| Solução | Quando Usar |
|---------|-------------|
| **The Graph** | Produção, queries complexas, múltiplos contratos |
| **Custom Indexer** | Dados proprietários, controle total |
| **Alchemy/Moralis APIs** | Prototipagem rápida, NFTs |
| **Event Listeners** | Real-time updates, aplicação específica |
| **Caching** | Dados que mudam pouco |

---

## 14.2 The Graph - Subgraphs

### O Que É The Graph

**Analogia**: The Graph é como criar um **índice de database** para blockchain.

```
Blockchain (Fonte)
    ↓ Events
Subgraph (Indexador)
    ↓ Transform & Store
GraphQL API (Query)
    ↓
Frontend
```

### Arquitetura

```
┌─────────────────────────────────────┐
│    Ethereum Blockchain              │
│  (Events: Transfer, Swap, etc.)     │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│     Graph Node (Indexer)            │
│  - Escuta events                    │
│  - Executa handlers (TypeScript)    │
│  - Armazena em PostgreSQL           │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│    GraphQL API                      │
│  { users { id, balance } }          │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│       Frontend                      │
│   (Query via Apollo/URQL)           │
└─────────────────────────────────────┘
```

### Componentes de um Subgraph

1. **Schema (GraphQL)**: Define entidades (como SQL tables)
2. **Manifest (subgraph.yaml)**: Configura contratos e events
3. **Mappings (AssemblyScript)**: Transforma events em entidades

---

## 14.3 Criando um Subgraph

### Setup

```bash
# Install Graph CLI
npm install -g @graphprotocol/graph-cli

# Create subgraph
graph init --product subgraph-studio my-subgraph

# Navigate
cd my-subgraph
```

### 1. Schema (schema.graphql)

```graphql
# Define entities (como SQL tables)
type Token @entity {
  id: ID!                    # Token address
  name: String!
  symbol: String!
  decimals: Int!
  totalSupply: BigInt!
}

type User @entity {
  id: ID!                    # User address
  balance: BigInt!
  transfersFrom: [Transfer!]! @derivedFrom(field: "from")
  transfersTo: [Transfer!]!   @derivedFrom(field: "to")
}

type Transfer @entity {
  id: ID!                    # Transaction hash + log index
  from: User!
  to: User!
  amount: BigInt!
  timestamp: BigInt!
  blockNumber: BigInt!
  transactionHash: String!
}
```

### 2. Manifest (subgraph.yaml)

```yaml
specVersion: 0.0.5
schema:
  file: ./schema.graphql

dataSources:
  - kind: ethereum/contract
    name: MyToken
    network: mainnet
    source:
      address: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48" # USDC
      abi: ERC20
      startBlock: 6082465 # Block do deploy
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Token
        - User
        - Transfer
      abis:
        - name: ERC20
          file: ./abis/ERC20.json
      eventHandlers:
        - event: Transfer(indexed address,indexed address,uint256)
          handler: handleTransfer
      file: ./src/mapping.ts
```

### 3. Mappings (src/mapping.ts)

```typescript
import { Transfer as TransferEvent } from "../generated/MyToken/ERC20"
import { Token, User, Transfer } from "../generated/schema"
import { BigInt } from "@graphprotocol/graph-ts"

// Handler para evento Transfer
export function handleTransfer(event: TransferEvent): void {
  // 1. Load ou create Token entity
  let token = Token.load(event.address.toHexString())
  if (token == null) {
    token = new Token(event.address.toHexString())
    token.name = "USD Coin"
    token.symbol = "USDC"
    token.decimals = 6
    token.totalSupply = BigInt.fromI32(0)
  }

  // 2. Load ou create FROM user
  let fromUser = User.load(event.params.from.toHexString())
  if (fromUser == null) {
    fromUser = new User(event.params.from.toHexString())
    fromUser.balance = BigInt.fromI32(0)
  }

  // 3. Load ou create TO user
  let toUser = User.load(event.params.to.toHexString())
  if (toUser == null) {
    toUser = new User(event.params.to.toHexString())
    toUser.balance = BigInt.fromI32(0)
  }

  // 4. Update balances
  fromUser.balance = fromUser.balance.minus(event.params.value)
  toUser.balance = toUser.balance.plus(event.params.value)

  // 5. Create Transfer entity
  let transfer = new Transfer(
    event.transaction.hash.toHexString() + "-" + event.logIndex.toString()
  )
  transfer.from = fromUser.id
  transfer.to = toUser.id
  transfer.amount = event.params.value
  transfer.timestamp = event.block.timestamp
  transfer.blockNumber = event.block.number
  transfer.transactionHash = event.transaction.hash.toHexString()

  // 6. Save entities
  token.save()
  fromUser.save()
  toUser.save()
  transfer.save()
}
```

### Deploy

```bash
# 1. Codegen (gera TypeScript types)
graph codegen

# 2. Build
graph build

# 3. Criar subgraph no Subgraph Studio (https://thegraph.com/studio/)
graph auth --studio <DEPLOY_KEY>

# 4. Deploy
graph deploy --studio my-subgraph
```

💡 **Pro Tip**: Após deploy, The Graph indexa desde `startBlock`. Pode demorar horas para contratos antigos!

---

## 14.4 GraphQL Queries

### Query Básicas

```graphql
# Buscar user específico
{
  user(id: "0x742d35cc6634c0532925a3b844bc9e7595f0beb") {
    id
    balance
  }
}

# Buscar múltiplos users (paginação)
{
  users(first: 10, skip: 0, orderBy: balance, orderDirection: desc) {
    id
    balance
  }
}

# Buscar transfers de um user
{
  transfers(
    where: { from: "0x742d35cc6634c0532925a3b844bc9e7595f0beb" }
    first: 100
    orderBy: timestamp
    orderDirection: desc
  ) {
    id
    to {
      id
    }
    amount
    timestamp
  }
}
```

### Filtering

```graphql
# Transfers > 1000 USDC
{
  transfers(where: { amount_gt: "1000000000" }) { # 1000 * 10^6
    from { id }
    to { id }
    amount
  }
}

# Transfers nos últimos 7 dias
{
  transfers(
    where: { timestamp_gt: "1704067200" } # Unix timestamp
    orderBy: timestamp
    orderDirection: desc
  ) {
    from { id }
    to { id }
    amount
    timestamp
  }
}

# Transfers entre dois addresses
{
  transfers(
    where: {
      from: "0xAlice"
      to: "0xBob"
    }
  ) {
    amount
    timestamp
  }
}
```

### Agregações

```graphql
# Total de transfers de um user
{
  user(id: "0x...") {
    transfersFrom(first: 1000) {
      amount
    }
    transfersTo(first: 1000) {
      amount
    }
  }
}
```

⚠️ **Limitação**: The Graph não suporta agregações nativas (SUM, AVG, etc.). Fazer client-side.

### Frontend com Apollo Client

```typescript
// Setup Apollo
import { ApolloClient, InMemoryCache, gql } from '@apollo/client';

const client = new ApolloClient({
  uri: 'https://api.thegraph.com/subgraphs/name/your-username/your-subgraph',
  cache: new InMemoryCache()
});

// Query
const GET_USERS = gql`
  query GetUsers {
    users(first: 10, orderBy: balance, orderDirection: desc) {
      id
      balance
    }
  }
`;

// React component
import { useQuery } from '@apollo/client';

function TopUsers() {
  const { loading, error, data } = useQuery(GET_USERS);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <ul>
      {data.users.map((user: any) => (
        <li key={user.id}>
          {user.id}: {user.balance} USDC
        </li>
      ))}
    </ul>
  );
}
```

---

## 14.5 Event Listeners Customizados

### Quando Usar

✅ **Use custom listeners quando**:
- Real-time updates (não quer esperar The Graph indexar)
- Aplicação específica (não precisa de queries complexas)
- Dados proprietários (não públicos)

### Implementação: Node.js Listener

```javascript
// indexer.js
const { ethers } = require('ethers');
const { Pool } = require('pg'); // PostgreSQL

// Setup
const provider = new ethers.JsonRpcProvider('YOUR_RPC_URL');
const pool = new Pool({
  connectionString: 'postgresql://user:password@localhost:5432/mydb'
});

const ERC20_ABI = [
  "event Transfer(address indexed from, address indexed to, uint256 value)"
];

const USDC_ADDRESS = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';

// Create database schema
async function setupDatabase() {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS transfers (
      id SERIAL PRIMARY KEY,
      from_address VARCHAR(42) NOT NULL,
      to_address VARCHAR(42) NOT NULL,
      amount NUMERIC NOT NULL,
      block_number INTEGER NOT NULL,
      transaction_hash VARCHAR(66) NOT NULL,
      timestamp INTEGER NOT NULL,
      created_at TIMESTAMP DEFAULT NOW()
    );

    CREATE INDEX IF NOT EXISTS idx_from ON transfers(from_address);
    CREATE INDEX IF NOT EXISTS idx_to ON transfers(to_address);
    CREATE INDEX IF NOT EXISTS idx_block ON transfers(block_number);
  `);
}

// Listen to events
async function listenToTransfers() {
  const contract = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, provider);

  console.log('Listening to USDC transfers...');

  contract.on('Transfer', async (from, to, amount, event) => {
    try {
      const block = await event.getBlock();

      await pool.query(
        `INSERT INTO transfers (from_address, to_address, amount, block_number, transaction_hash, timestamp)
         VALUES ($1, $2, $3, $4, $5, $6)`,
        [
          from,
          to,
          amount.toString(),
          event.blockNumber,
          event.transactionHash,
          block.timestamp
        ]
      );

      console.log(`Indexed transfer: ${from} → ${to}: ${ethers.formatUnits(amount, 6)} USDC`);
    } catch (error) {
      console.error('Error indexing transfer:', error);
    }
  });
}

// Backfill histórico
async function backfill(fromBlock, toBlock) {
  const contract = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, provider);

  console.log(`Backfilling blocks ${fromBlock} to ${toBlock}...`);

  const filter = contract.filters.Transfer();
  const events = await contract.queryFilter(filter, fromBlock, toBlock);

  for (const event of events) {
    const block = await event.getBlock();

    await pool.query(
      `INSERT INTO transfers (from_address, to_address, amount, block_number, transaction_hash, timestamp)
       VALUES ($1, $2, $3, $4, $5, $6)
       ON CONFLICT DO NOTHING`,
      [
        event.args.from,
        event.args.to,
        event.args.value.toString(),
        event.blockNumber,
        event.transactionHash,
        block.timestamp
      ]
    );
  }

  console.log(`Backfilled ${events.length} transfers`);
}

// API para query
async function getTransfersByUser(address, limit = 100) {
  const result = await pool.query(
    `SELECT * FROM transfers
     WHERE from_address = $1 OR to_address = $1
     ORDER BY block_number DESC
     LIMIT $2`,
    [address, limit]
  );

  return result.rows;
}

// Main
async function main() {
  await setupDatabase();

  // Backfill últimos 1000 blocos
  const currentBlock = await provider.getBlockNumber();
  await backfill(currentBlock - 1000, currentBlock);

  // Listen real-time
  await listenToTransfers();
}

main();
```

### Express API

```javascript
// api.js
const express = require('express');
const { Pool } = require('pg');

const app = express();
const pool = new Pool({ connectionString: 'postgresql://...' });

// Get transfers by user
app.get('/transfers/:address', async (req, res) => {
  const { address } = req.params;
  const { limit = 100, offset = 0 } = req.query;

  const result = await pool.query(
    `SELECT * FROM transfers
     WHERE from_address = $1 OR to_address = $1
     ORDER BY block_number DESC
     LIMIT $2 OFFSET $3`,
    [address, limit, offset]
  );

  res.json({
    data: result.rows,
    total: result.rowCount
  });
});

// Get total volume by user
app.get('/volume/:address', async (req, res) => {
  const { address } = req.params;

  const result = await pool.query(
    `SELECT
       SUM(CASE WHEN from_address = $1 THEN amount ELSE 0 END) as sent,
       SUM(CASE WHEN to_address = $1 THEN amount ELSE 0 END) as received
     FROM transfers
     WHERE from_address = $1 OR to_address = $1`,
    [address]
  );

  res.json(result.rows[0]);
});

app.listen(3000, () => console.log('API running on port 3000'));
```

---

## 14.6 Alternativas ao The Graph

### 1. Alchemy NFT API

**Quando usar**: NFTs, prototipagem rápida

```javascript
const { Alchemy, Network } = require('alchemy-sdk');

const alchemy = new Alchemy({
  apiKey: 'YOUR_API_KEY',
  network: Network.ETH_MAINNET
});

// Get NFTs owned by address
async function getNFTs(address) {
  const nfts = await alchemy.nft.getNftsForOwner(address);
  return nfts.ownedNfts;
}

// Get NFT metadata
async function getNFTMetadata(contractAddress, tokenId) {
  const metadata = await alchemy.nft.getNftMetadata(contractAddress, tokenId);
  return metadata;
}

// Get NFT sales
async function getNFTSales(contractAddress) {
  const sales = await alchemy.nft.getNftSales({
    contractAddress: contractAddress,
    fromBlock: 'latest'
  });
  return sales;
}
```

### 2. Moralis

**Quando usar**: Multi-chain, prototipagem

```javascript
const Moralis = require('moralis').default;

await Moralis.start({
  apiKey: 'YOUR_API_KEY'
});

// Get token balances
const balances = await Moralis.EvmApi.token.getWalletTokenBalances({
  address: '0x...',
  chain: '0x1'
});

// Get transaction history
const transactions = await Moralis.EvmApi.transaction.getWalletTransactions({
  address: '0x...',
  chain: '0x1'
});

// Get NFTs
const nfts = await Moralis.EvmApi.nft.getWalletNFTs({
  address: '0x...',
  chain: '0x1'
});
```

### 3. Covalent

**Quando usar**: Multi-chain, dados históricos

```javascript
// RESTful API
const response = await fetch(
  `https://api.covalenthq.com/v1/1/address/${address}/balances_v2/?key=YOUR_API_KEY`
);
const data = await response.json();
```

---

## 14.7 Quando Usar Cada Solução

| Solução | Prós | Contras | Quando Usar |
|---------|------|---------|-------------|
| **The Graph** | ✅ Descentralizado<br>✅ GraphQL<br>✅ Custom entities | ❌ Tempo de setup<br>❌ Delay de indexing | Produção, queries complexas |
| **Custom Indexer** | ✅ Controle total<br>✅ SQL nativo | ❌ Infraestrutura<br>❌ Manutenção | Dados proprietários |
| **Alchemy/Moralis** | ✅ Setup rápido<br>✅ APIs prontas | ❌ Centralizado<br>❌ Custo | Prototipagem, NFTs |
| **Event Listeners** | ✅ Real-time<br>✅ Simples | ❌ Não persiste<br>❌ Sem histórico | Updates live |

### Decision Tree

```
Precisa de dados históricos?
├─ Sim
│  ├─ Queries complexas (JOINs, filters)?
│  │  ├─ Sim → The Graph
│  │  └─ Não → Alchemy/Moralis APIs
│  └─ Dados proprietários?
│     └─ Sim → Custom Indexer
└─ Não (apenas real-time)
   └─ Event Listeners
```

---

## 📖 Glossário

**Indexing**
> Processar e armazenar dados blockchain para queries rápidas.
> **Analogia**: Como criar índices em database SQL.

**Subgraph**
> Definição de como indexar dados de smart contract (schema + handlers).
> **Analogia**: Como migration + seed script de database.

**Entity**
> Objeto armazenado (como table row em SQL).
> **Exemplo**: User, Transfer, Token.

**Mapping**
> Código que transforma events em entities.
> **Analogia**: ETL (Extract, Transform, Load).

**Event Listener**
> Código que escuta eventos blockchain em tempo real.
> **Uso**: Updates live, notificações.

**Backfill**
> Indexar dados históricos (blocos antigos).
> **Quando**: Após deploy de subgraph ou custom indexer.

**GraphQL**
> Query language para APIs.
> **Vantagem**: Client especifica exatamente o que quer.

---

## 🔒 Best Practices

### Subgraphs
- [ ] Começar com `startBlock` para evitar indexar desde genesis
- [ ] Usar IDs compostos para entidades únicas (`tx-hash-log-index`)
- [ ] Evitar lógica complexa em handlers (manter simples)
- [ ] Adicionar indexes em campos filtráveis
- [ ] Testar localmente antes de deploy (graph-node local)

### Custom Indexers
- [ ] Idempotência (INSERT ON CONFLICT DO NOTHING)
- [ ] Indexes em colunas filtradas (from_address, to_address)
- [ ] Pagination em queries (LIMIT + OFFSET)
- [ ] Rate limiting no RPC provider
- [ ] Monitoramento de lag (current block vs indexed block)

### Queries
- [ ] Pagination (first + skip)
- [ ] Avoid fetching demais (max 1000 por query)
- [ ] Cache em frontend (Apollo cache)
- [ ] Error handling robusto

---

## 📝 Exercícios

### Exercício 1: DEX Subgraph

**Objetivo**: Criar subgraph para DEX do Cap 10.

**Entities**:
- Token
- User
- Swap
- LiquidityPool

**Events**:
- Swap
- AddLiquidity
- RemoveLiquidity

**Queries**:
- Top traders (por volume)
- Pool history (TVL ao longo do tempo)
- User portfolio

---

### Exercício 2: NFT Indexer

**Objetivo**: Custom indexer para NFT collection.

**Features**:
- Track mints, transfers, sales
- Floor price tracking
- Rarity metadata
- Owner analytics

**Tech**: Node.js + PostgreSQL + Express

---

## 📚 Recursos

### The Graph
1. **[The Graph Docs](https://thegraph.com/docs/)**
2. **[Subgraph Studio](https://thegraph.com/studio/)**
3. **[AssemblyScript Docs](https://www.assemblyscript.org/)**

### Ferramentas
- **[Graph CLI](https://www.npmjs.com/package/@graphprotocol/graph-cli)**
- **[Graph Node](https://github.com/graphprotocol/graph-node)** (local)
- **[Apollo Client](https://www.apollographql.com/docs/react/)**

---

## 🎯 Próximos Passos

**Você agora sabe**:
- ✅ Por que indexing é necessário
- ✅ Criar subgraphs (The Graph)
- ✅ GraphQL queries
- ✅ Custom indexers
- ✅ Event listeners
- ✅ Quando usar cada solução

**Próximo Capítulo**: [Capítulo 15 - Backend](./EBOOK_CAPITULO_15_BACKEND.md)
- Node.js + Ethers.js
- Webhooks para eventos
- IPFS pinning
- Arquitetura híbrida (on-chain + off-chain)

**Stack Completo**:
- Cap 10 (DEX) + Cap 13 (Frontend) + Cap 14 (Indexing) = DEX completo!
- Subgraph para queries + Frontend para UI + Smart contract

💡 **Projeto**: Uniswap clone com subgraph para analytics dashboard.

---

**Autor**: Baseado em roadmap ITA Blockchain Club + The Graph docs
**Última Atualização**: 2025-11-14
