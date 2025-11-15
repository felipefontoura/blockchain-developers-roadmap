# Apêndice B: Glossário Técnico Completo

## Introdução

**Este glossário é sua referência rápida para todos os termos Web3 mencionados no ebook.**

**Como usar:**
- 📖 **Ordem alfabética** (A-Z)
- 🔍 **Ctrl+F** para buscar termos específicos
- 🌐 **Comparações Web2** quando relevante
- 💡 **Exemplos práticos** para conceitos complexos

**Convenções:**
- **Bold**: Termo principal
- *Itálico*: Termos relacionados (veja também)
- 🔗: Links para capítulos do ebook
- ⚠️: Conceitos de segurança
- 💰: Relacionado a custos/economia

---

## A

### **ABI (Application Binary Interface)**
Interface que define como interagir com smart contract compilado. Contém assinaturas de funções, eventos, tipos de dados.

**Web2 equivalente**: API schema (OpenAPI/Swagger)

**Exemplo**:
```json
[
  {
    "type": "function",
    "name": "transfer",
    "inputs": [
      {"name": "to", "type": "address"},
      {"name": "amount", "type": "uint256"}
    ],
    "outputs": [{"type": "bool"}]
  }
]
```

*Ver também*: Bytecode, Contract Interface

---

### **Abstract Contract**
Contrato Solidity que não pode ser deployed diretamente. Usado como base class.

**Exemplo**:
```solidity
abstract contract Ownable {
    function owner() public view virtual returns (address);
}
```

*Ver também*: Interface, Inheritance

---

### **Account**
Entidade na blockchain que pode enviar transações e ter saldo.

**Tipos**:
- **EOA** (Externally Owned Account): Controlada por private key
- **Contract Account**: Smart contract (controlado por código)

**Web2 equivalente**: User account vs Service account

*Ver também*: Address, EOA, Smart Contract

---

### **Account Abstraction (ERC-4337)**
Permite que smart contracts atuem como wallets, removendo necessidade de EOAs.

**Benefícios**:
- Gasless transactions (sponsor pode pagar gas)
- Social recovery (recuperar wallet sem seed phrase)
- Batching (múltiplas ops em 1 tx)

**Status**: Live na mainnet (2023+)

*Ver também*: Smart Wallet, Paymaster, UserOperation

---

### **Address**
Identificador único de 20 bytes (40 hex chars) para account ou contract.

**Formato**: `0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb`

**Web2 equivalente**: Username, UUID

**Tipos**:
- EOA address: Derivado de public key
- Contract address: Calculado de deployer address + nonce

*Ver também*: Account, Checksum Address

---

### **Airdrop**
Distribuição gratuita de tokens para wallets (geralmente para marketing/comunidade).

**Web2 equivalente**: Free trial, promotional credits

⚠️ **Segurança**: Desconfie de airdrops que pedem approve de tokens existentes.

---

### **Alpha (DeFi slang)**
Informação valiosa/estratégia lucrativa ainda não conhecida pelo mercado.

**Exemplo**: "Achei um alpha: novo pool de staking com 200% APY"

**Web2 equivalente**: Inside information, competitive advantage

---

### **AMM (Automated Market Maker)**
Protocolo DeFi que usa fórmula matemática (não orderbook) para fazer trades.

**Fórmula comum**: `x * y = k` (constant product)

**Exemplos**: Uniswap, SushiSwap, PancakeSwap

**Web2 equivalente**: Market maker tradicional (mas automatizado)

🔗 *Ver Capítulo 10: DeFi Primitives*

*Ver também*: DEX, Liquidity Pool, Slippage

---

### **Anchor (Solana)**
Framework para desenvolvimento Solana (similar ao Hardhat/Foundry para Ethereum).

**Linguagem**: Rust

*Ver também*: Solana, Rust, Program (Solana)

---

### **Anvil**
Local blockchain node (parte do Foundry) para testes.

**Web2 equivalente**: Localhost/Docker para dev

**Comando**: `anvil` (inicia node local)

*Ver também*: Foundry, Ganache, Hardhat Network

---

### **APY (Annual Percentage Yield)**
Retorno anual incluindo compound.

💰 **Fórmula**: `APY = (1 + r/n)^n - 1`
- r = taxa de juros
- n = frequência de compound

**Exemplo**: 100% APR com daily compound = 171% APY

*Ver também*: APR, Staking, Yield Farming

---

### **APR (Annual Percentage Rate)**
Retorno anual SEM compound.

**Diferença do APY**: APR não considera reinvestimento de rewards.

---

### **Arbitrage**
Lucrar com diferença de preço do mesmo asset em mercados diferentes.

**Exemplo**: Comprar ETH por $2000 na Uniswap, vender por $2010 na SushiSwap.

**Web2 equivalente**: Arbitragem em mercados financeiros tradicionais

⚠️ **MEV**: Bots competem por oportunidades de arbitragem.

*Ver também*: MEV, Sandwich Attack, Flash Loan

---

### **Arbitrum**
Layer 2 da Ethereum usando Optimistic Rollup.

**TPS**: ~4000
**Custo**: $0.01-0.50/tx
**EVM**: 100% compatível

🔗 *Ver Apêndice A: Comparativo de Blockchains*

*Ver também*: Optimistic Rollup, Layer 2, Optimism

---

### **Attestation**
Declaração criptograficamente assinada (prova de algo).

**Exemplos**:
- Prova de identidade (KYC)
- Prova de participação (POAP)
- Validator attestation (Ethereum consensus)

*Ver também*: Oracle, Signature, Proof

---

### **Audit**
Revisão de segurança profissional de smart contracts.

**Custo**: $5k-500k+ (depende da complexidade)

**Firmas conhecidas**: Trail of Bits, OpenZeppelin, ConsenSys Diligence

🔗 *Ver Capítulo 17: Auditoria*

⚠️ **Importante**: Audit ≠ garantia de segurança (mas reduz riscos drasticamente)

*Ver também*: Formal Verification, Bug Bounty, Slither

---

### **AVM (Algorand Virtual Machine)**
Virtual machine da blockchain Algorand.

*Ver também*: EVM, SVM (Solana), WASM

---

## B

### **Backrunning**
MEV strategy: Executar transação imediatamente APÓS outra para lucrar.

**Exemplo**: User compra token → preço sobe → bot vende token

*Ver também*: MEV, Frontrunning, Sandwich Attack

---

### **Base**
Layer 2 da Ethereum desenvolvido pela Coinbase (usa OP Stack).

**Chain ID**: 8453
**EVM**: 100% compatível

*Ver também*: Optimism, OP Stack, Layer 2

---

### **Batch Transaction**
Agrupar múltiplas transações em uma única (economiza gas).

**Exemplo**: Aprovar + transferir em 1 tx

**Implementação**: Multicall, ERC-4337 UserOp

*Ver também*: Multicall, Account Abstraction

---

### **BFT (Byzantine Fault Tolerance)**
Capacidade de sistema atingir consenso mesmo com nós maliciosos (até 33%).

**Web2 equivalente**: Fault tolerance em distributed systems

**Usado em**: Tendermint (Cosmos), PBFT

*Ver também*: Consensus, Validator, Slashing

---

### **Block**
Conjunto de transações agrupadas e adicionadas à blockchain.

**Componentes**:
- Header (hash anterior, timestamp, nonce)
- Transactions
- State root

**Ethereum**: ~12s por block
**Bitcoin**: ~10 min por block

*Ver também*: Blockchain, Mining, Validator

---

### **Block Explorer**
Website para visualizar transações/blocks/contracts.

**Exemplos**:
- Etherscan (Ethereum)
- Arbiscan (Arbitrum)
- Solscan (Solana)

**Web2 equivalente**: Database viewer, logs dashboard

---

### **Block Number**
Altura da blockchain (quantos blocks desde genesis).

**Solidity**: `block.number`

**Uso**: Timelock (executar após X blocks)

⚠️ **L2 gotcha**: Block number incrementa mais rápido que L1

---

### **Blockchain**
Ledger distribuído e imutável de transações, organizado em blocos encadeados.

**Web2 equivalente**: Database distribuída, mas imutável

**Propriedades**:
- Descentralizado (sem single point of failure)
- Imutável (não pode alterar histórico)
- Transparente (todos veem transações)

🔗 *Ver Capítulo 1: Blockchain para Desenvolvedores*

---

### **Block Reward**
Recompensa para validator/miner por criar block.

**Ethereum pós-merge**: ~0.02 ETH/block (varia)
**Bitcoin**: 3.125 BTC (halving em 2024)

💰 *Ver também*: Mining, Validator, Staking

---

### **BNB Chain**
Blockchain EVM-compatible (anteriormente Binance Smart Chain).

**TPS**: ~2000
**Consensus**: PoSA (Proof of Staked Authority)

*Ver também*: EVM, Binance

---

### **Bridge**
Protocolo que permite transferir assets entre blockchains diferentes.

**Tipos**:
- **Trusted**: Custodial (ex: Binance Bridge)
- **Trustless**: Lock & mint (ex: Wormhole, LayerZero)

⚠️ **Segurança**: Bridges são alvos comuns de hacks ($2B+ roubado em 2022)

**Exemplos**:
- Ethereum ↔ Arbitrum: Official bridge
- Ethereum ↔ Solana: Wormhole

*Ver também*: Cross-chain, Wrapped Tokens, LayerZero

---

### **Bug Bounty**
Programa que paga hackers éticos por encontrar vulnerabilidades.

**Platforms**: Immunefi, Code4rena, HackerOne

💰 **Payouts**: $1k-10M (Immunefi teve bounty de $10M)

🔗 *Ver Capítulo 17: Auditoria*

*Ver também*: Audit, Whitehack, Exploit

---

### **Burn**
Remover tokens de circulação permanentemente (enviar para address 0x000...000).

**Efeito**: Reduz supply → potencialmente aumenta preço

**Exemplo**: ETH fee burn (EIP-1559)

*Ver também*: Mint, Total Supply, Deflationary

---

### **Bytecode**
Código compilado de smart contract que EVM executa.

**Formato**: Hex (ex: `0x6080604052...`)

**Web2 equivalente**: Assembly, machine code

**Gerado por**: `solc` (Solidity compiler), `vyper`

*Ver também*: Opcode, EVM, ABI

---

## C

### **Calldata**
Dados enviados com transação (input para função de contract).

**Custo**: 4 gas/byte (zero), 16 gas/byte (non-zero)

**Otimização**: Comprimir calldata economiza gas

**Solidity**: `msg.data`

*Ver também*: Gas Optimization, Transactions

---

### **Cast**
Ferramenta CLI do Foundry para interagir com blockchain.

**Exemplos**:
```bash
cast call <address> "balanceOf(address)" <user>
cast send <address> "transfer(address,uint256)" <to> 100
```

*Ver também*: Foundry, Forge, Anvil

---

### **CEX (Centralized Exchange)**
Exchange centralizada (custodiada por empresa).

**Exemplos**: Binance, Coinbase, Kraken

**Web2 equivalente**: Banco, corretora tradicional

*Ver também*: DEX, Custody, KYC

---

### **Chain ID**
Identificador único de blockchain.

**Exemplos**:
- 1: Ethereum Mainnet
- 42161: Arbitrum
- 10: Optimism
- 8453: Base

**Uso**: Prevenir replay attacks entre chains

**Solidity**: `block.chainid`

*Ver também*: Network, Replay Attack

---

### **Chainlink**
Rede descentralizada de oracles.

**Serviços**:
- Price Feeds (preços de assets)
- VRF (randomness verificável)
- Automation (Keepers)
- CCIP (cross-chain messaging)

🔗 *Ver Capítulo 11: Oracles*

*Ver também*: Oracle, Oracle Problem, Pyth

---

### **Checksum Address**
Address com capitalização específica para detectar erros.

**Exemplo**:
- ❌ `0xd8da6bf26964af9d7eed9e03e53415d37aa96045`
- ✅ `0xD8dA6BF26964aF9D7eEd9e03E53415D37aA96045` (checksum)

**Validação**: EIP-55 (hash do address define maiúsculas)

*Ver também*: Address, EIP-55

---

### **Clarity**
Linguagem de smart contracts da blockchain Stacks (Bitcoin L2).

**Características**:
- Decidable (não-Turing complete)
- Lisp-like syntax
- Readable on-chain

*Ver também*: Stacks, Bitcoin, Solidity

---

### **Collateral**
Asset depositado como garantia (em lending, por exemplo).

**Exemplo**: Depositar 1000 USDC para tomar emprestado 500 DAI

💰 **Collateralization ratio**: 200% neste exemplo

*Ver também*: Lending, Liquidation, Overcollateralized

---

### **Compiler**
Converte código Solidity em bytecode EVM.

**Exemplos**:
- `solc` (Solidity compiler)
- `vyper` (Vyper compiler)

**Versão**: Especificar em contrato (`pragma solidity ^0.8.20;`)

*Ver também*: Bytecode, Opcode, Solidity

---

### **Composability**
Capacidade de combinar protocolos DeFi como LEGO blocks.

**Exemplo**: Uniswap LP token → usar como collateral no Aave

**Web2 equivalente**: Microservices, APIs

**Vantagem DeFi**: Permissionless (não precisa de integração manual)

⚠️ **Risco**: Composability = superfície de ataque maior

*Ver também*: DeFi, Protocol, Interoperability

---

### **Consensus**
Mecanismo para nodes concordarem sobre estado da blockchain.

**Tipos**:
- **PoW** (Proof of Work): Mineração
- **PoS** (Proof of Stake): Staking
- **PoA** (Proof of Authority): Validators aprovados
- **PBFT**: Byzantine fault tolerance

🔗 *Ver Capítulo 1: Blockchain Fundamentals*

*Ver também*: Mining, Validator, Byzantine Fault Tolerance

---

### **Constant Product Formula**
Fórmula AMM: `x * y = k`

**Exemplo**: Pool com 100 ETH e 200k USDC
- `k = 100 * 200000 = 20000000`
- Trade 10 ETH → recebe `200000 - (20000000 / 110) = 18182 USDC`

🔗 *Ver Capítulo 10: DeFi Primitives*

*Ver também*: AMM, Uniswap, Slippage

---

### **Constructor**
Função especial executada apenas no deploy do contrato.

**Solidity**:
```solidity
constructor(address _owner) {
    owner = _owner;
}
```

**Característica**: Não pode ser chamada depois do deploy

*Ver também*: Deployment, Initializer (proxies)

---

### **Contract**
→ Ver **Smart Contract**

---

### **Contract Account**
→ Ver **Account**

---

### **Coverage**
Métrica de quanto do código é testado.

**Ferramentas**:
- `forge coverage` (Foundry)
- `hardhat-coverage` (Hardhat)

**Meta**: 90%+ coverage antes de produção

🔗 *Ver Capítulo 6: Testing*

*Ver também*: Testing, CI/CD

---

### **Create2**
Opcode que permite deploy de contrato em address determinístico.

**Uso**: Counterfactual contracts (address conhecido antes de deploy)

**Fórmula**: `keccak256(0xff, sender, salt, bytecode)`

**Exemplo**: Uniswap v3 pools (address previsível)

*Ver também*: Opcode, Deployment, Deterministic

---

### **Cross-chain**
Interação entre blockchains diferentes.

**Métodos**:
- Bridges (lock & mint)
- Oracles (Chainlink CCIP)
- Native (IBC no Cosmos)

*Ver também*: Bridge, Interoperability, LayerZero

---

### **Custody**
Guarda de assets (quem controla private keys).

**Tipos**:
- **Self-custody**: Você controla keys (wallet própria)
- **Custodial**: Exchange/empresa controla (Coinbase, Binance)

⚠️ **"Not your keys, not your coins"**

*Ver também*: Wallet, Private Key, CEX

---

## D

### **DAO (Decentralized Autonomous Organization)**
Organização governada por smart contracts e votação on-chain.

**Componentes**:
- Governance token (voto)
- Treasury (fundos)
- Proposals (sugestões)
- Timelock (delay para executar)

**Exemplos**: MakerDAO, Uniswap DAO, ENS DAO

🔗 *Ver Capítulo 12: Upgradeable Contracts*

*Ver também*: Governance, Voting, Timelock

---

### **DApp (Decentralized Application)**
Aplicação que usa smart contracts no backend.

**Componentes**:
- Frontend (React, Vue, etc)
- Smart contracts (backend)
- Wallet (autenticação)
- RPC provider (comunicação)

**Web2 equivalente**: Web app, mas backend descentralizado

🔗 *Ver Capítulo 13: Frontend Integration*

*Ver também*: Smart Contract, Frontend, Web3.js

---

### **DAI**
Stablecoin descentralizada (mantida por MakerDAO).

**Collateral**: ETH, WBTC, USDC (overcollateralized)

**Valor**: $1 (mantido por mecanismo de arbitragem)

*Ver também*: Stablecoin, MakerDAO, Collateral

---

### **Decimals**
Casas decimais de token.

**Padrão**: 18 (como ETH: 1 ETH = 10^18 wei)

**Exceções**: USDC/USDT = 6 decimals

**Solidity**:
```solidity
function decimals() public view returns (uint8) {
    return 18;
}
```

⚠️ **Cuidado**: Sempre verificar decimals antes de cálculos

*Ver também*: Wei, Token, ERC-20

---

### **DeFi (Decentralized Finance)**
Sistema financeiro sem intermediários (bancos, corretoras).

**Primitives**:
- DEX (trading)
- Lending (empréstimos)
- Staking (rewards)
- Derivatives (futuros, opções)

**TVL total**: $50B+ (2024)

🔗 *Ver Capítulo 10: DeFi Primitives*

*Ver também*: AMM, Lending, Yield Farming

---

### **Delegatecall**
Opcode que executa código de outro contrato MAS mantém contexto atual.

**Uso**: Proxies, libraries

⚠️ **Perigo**: Storage collision, security bugs

**Exemplo**:
```solidity
(bool success, ) = logic.delegatecall(msg.data);
```

🔗 *Ver Capítulo 8: Security Vulnerabilities*

*Ver também*: Proxy Pattern, Storage Collision, Opcode

---

### **Deployment**
Processo de publicar smart contract na blockchain.

**Custo**: Depende do tamanho (bytecode)

**Limite**: 24KB por contrato (EIP-170)

**Ferramentas**: Foundry, Hardhat, Remix

🔗 *Ver Capítulo 18: Deployment*

*Ver também*: Create2, Bytecode, Constructor

---

### **DePIN (Decentralized Physical Infrastructure Network)**
Redes descentralizadas de infraestrutura física.

**Exemplos**:
- Helium (wireless)
- Filecoin (storage)
- Render Network (GPU rendering)

**Blockchain comum**: Solana (throughput alto)

*Ver também*: Solana, IoT

---

### **Deterministic**
Sistema onde mesmo input sempre gera mesmo output.

**Blockchain**: Determinística (todos nodes chegam ao mesmo state)

**Exemplo**: `keccak256("hello")` sempre dá mesmo hash

*Ver também*: Consensus, EVM, Randomness

---

### **DEX (Decentralized Exchange)**
Exchange descentralizada (não-custodial).

**Tipos**:
- **AMM**: Uniswap, SushiSwap (constant product)
- **Orderbook**: dYdX, Serum (Solana)
- **Aggregator**: 1inch, Matcha (roteamento)

**Vantagem**: Self-custody, permissionless
**Desvantagem**: Slippage, MEV

*Ver também*: AMM, Liquidity Pool, CEX

---

### **Diamond Pattern**
Proxy pattern que permite múltiplos contratos de lógica (facets).

**Vantagem**: Contrato > 24KB (split em facets)

**Padrão**: EIP-2535

🔗 *Ver Capítulo 12: Upgradeable Contracts*

*Ver também*: Proxy, UUPS, Transparent Proxy

---

### **Difficulty**
Parâmetro que ajusta dificuldade de mineração (PoW).

**Bitcoin**: Ajusta a cada 2016 blocks (~2 semanas)

**Ethereum**: Não usa mais (migrou para PoS)

*Ver também*: Mining, PoW, Hash Rate

---

### **Dutch Auction**
Leilão onde preço começa alto e vai diminuindo até alguém comprar.

**Uso DeFi**:
- NFT launches
- Token sales
- Liquidations

**Vantagem**: Price discovery eficiente

*Ver também*: Auction, NFT, Price Discovery

---

## E

### **Echidna**
Fuzzer para smart contracts Solidity (property-based testing).

**Uso**: Encontrar edge cases que testes manuais não pegam

🔗 *Ver Capítulo 6: Testing*

*Ver também*: Fuzzing, Foundry Fuzz, Slither

---

### **EIP (Ethereum Improvement Proposal)**
Proposta de melhoria para Ethereum.

**Tipos**:
- **EIP**: Core protocol
- **ERC**: Application-level (tokens, etc)

**Exemplos importantes**:
- EIP-1559 (fee market)
- EIP-4337 (account abstraction)
- ERC-20 (token standard)
- ERC-721 (NFT standard)

*Ver também*: ERC, Governance, Hard Fork

---

### **EIP-1559**
Mecanismo de fee market do Ethereum (pós-London fork, 2021).

**Mudanças**:
- Base fee (queimada)
- Priority fee (tip para validator)
- Fee dinâmico (ajusta por demanda)

💰 **Efeito**: ETH deflacionário (burn > issuance)

*Ver também*: Gas, Base Fee, Priority Fee, Burn

---

### **EIP-2535**
→ Ver **Diamond Pattern**

---

### **EIP-4337**
→ Ver **Account Abstraction**

---

### **EIP-712**
Padrão para assinatura tipada de mensagens.

**Uso**: Signatures humano-legíveis (Permit, gasless txs)

**Vantagem**: Usuário vê o que está assinando (vs raw bytes)

*Ver também*: Signature, Permit, Gasless Transactions

---

### **Emit**
Keyword Solidity para emitir evento.

```solidity
emit Transfer(from, to, amount);
```

**Custo**: Gas (log storage)

**Uso**: Frontend escuta eventos para atualizar UI

*Ver também*: Event, Logs, Indexing

---

### **ENS (Ethereum Name Service)**
DNS descentralizado para Ethereum (mapeia nomes → addresses).

**Exemplo**: `vitalik.eth` → `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`

**Uso**: Addresses legíveis, resolver múltiplas chains

**Web2 equivalente**: DNS (domain names)

*Ver também*: Address, Domain, Resolver

---

### **EOA (Externally Owned Account)**
Account controlada por private key (não-contract).

**Características**:
- Pode iniciar transações
- Não tem código
- Controlada por humano (ou bot)

**Web2 equivalente**: User account

*Ver também*: Account, Contract Account, Private Key

---

### **ERC (Ethereum Request for Comments)**
Padrão de aplicação (subset de EIP).

**Principais ERCs**:
- **ERC-20**: Fungible tokens
- **ERC-721**: NFTs
- **ERC-1155**: Multi-token (fungible + NFT)
- **ERC-4626**: Tokenized vaults

🔗 *Ver Capítulo 9: Tokens*

*Ver também*: EIP, Token Standard

---

### **ERC-20**
Padrão de token fungível.

**Funções obrigatórias**:
- `totalSupply()`
- `balanceOf(address)`
- `transfer(address, uint256)`
- `approve(address, uint256)`
- `transferFrom(address, address, uint256)`

🔗 *Ver Capítulo 9: Tokens*

*Ver também*: Token, Fungible, ERC-721

---

### **ERC-721**
Padrão de NFT (non-fungible token).

**Diferença ERC-20**: Cada token é único (ID único)

**Uso**: Arte digital, colecionáveis, gaming items

🔗 *Ver Capítulo 9: Tokens*

*Ver também*: NFT, Token ID, Metadata

---

### **ERC-1155**
Padrão multi-token (combina fungible + non-fungible).

**Vantagem**: Múltiplos tipos de token em 1 contrato (economia de gas)

**Uso**: Gaming (100 swords + 1 unique armor)

*Ver também*: ERC-20, ERC-721, Gaming

---

### **ERC-4626**
Padrão de tokenized vault.

**Uso**: Yield farming, staking (LP token padronizado)

**Vantagem**: Composability entre vaults

🔗 *Ver Capítulo 10: DeFi Primitives*

*Ver também*: Vault, Yield, Staking

---

### **Ethers.js**
Biblioteca JavaScript para interagir com Ethereum.

**Versão atual**: v6 (2023+)

**Alternativa**: Web3.js (mais antiga)

🔗 *Ver Capítulo 13: Frontend Integration*

**Exemplo**:
```javascript
const provider = new ethers.JsonRpcProvider(RPC_URL);
const balance = await provider.getBalance(address);
```

*Ver também*: Web3.js, Provider, Signer

---

### **Etherscan**
Block explorer mais usado para Ethereum.

**URL**: etherscan.io

**Recursos**:
- Ver transações, blocks, addresses
- Ler contratos verified
- Write contract (interagir via UI)

**Alternativas**: Blockscan, Tenderly

*Ver também*: Block Explorer, Contract Verification

---

### **Event**
Log emitido por smart contract (armazenado fora do state).

**Uso**:
- Frontend tracking
- Off-chain indexing (The Graph)
- Audit trail

**Custo**: Mais barato que storage

**Solidity**:
```solidity
event Transfer(address indexed from, address indexed to, uint256 amount);
```

🔗 *Ver Capítulo 14: Indexing*

*Ver também*: Emit, Logs, Indexed

---

### **EVM (Ethereum Virtual Machine)**
Máquina virtual que executa smart contracts.

**Características**:
- Stack-based (256-bit words)
- Determinística
- Sandboxed (isolada)

🔗 *Ver Capítulo 2: Anatomia da EVM*

**Chains EVM-compatible**: Arbitrum, Polygon, BNB Chain, Avalanche

*Ver também*: Opcode, Bytecode, Gas

---

### **EVM-Compatible**
Blockchain que pode rodar bytecode Solidity (mesmo que Ethereum).

**Vantagem**: Porta código Solidity sem mudanças

**Exemplos**: Arbitrum, Optimism, Polygon, BNB Chain, Avalanche

*Ver também*: EVM, Solidity, Layer 2

---

### **Exploit**
Ataque que abusa de vulnerabilidade em contrato.

**Exemplos históricos**:
- DAO hack (reentrancy, $60M, 2016)
- Parity multisig (delegatecall, $280M, 2017)
- Poly Network (logic bug, $600M, 2021)

🔗 *Ver Capítulo 8: Security Vulnerabilities*

*Ver também*: Hack, Vulnerability, Audit

---

## F

### **Facet**
Contrato de lógica individual no Diamond pattern.

*Ver também*: Diamond Pattern, Proxy

---

### **Factory Pattern**
Contrato que cria outros contratos.

**Exemplo**: Uniswap factory cria pools

**Solidity**:
```solidity
contract Factory {
    function createPair(address tokenA, address tokenB) external returns (address) {
        return address(new Pair(tokenA, tokenB));
    }
}
```

🔗 *Ver Capítulo 5: Design Patterns*

*Ver também*: Design Pattern, Create2, Clone

---

### **Fallback Function**
Função executada quando chamada não corresponde a nenhuma função.

**Solidity**:
```solidity
fallback() external payable {
    // Recebe ETH ou chamadas desconhecidas
}
```

⚠️ **Segurança**: Limite gas (evitar reentrancy)

*Ver também*: Receive, Reentrancy, Gas Limit

---

### **Fee**
→ Ver **Gas Fee**, **Protocol Fee**

---

### **Finality**
Momento em que transação se torna irreversível.

**Tempos**:
- Ethereum: ~13 min (2 epochs)
- Solana: ~0.4s
- Bitcoin: ~60 min (6 confirmações)

**Tipos**:
- **Probabilistic**: Bitcoin, Ethereum PoW (quanto mais blocos, mais seguro)
- **Absolute**: Ethereum PoS, Tendermint (finality instantânea)

*Ver também*: Confirmation, Reorganization, Consensus

---

### **Flash Loan**
Empréstimo que deve ser pago na MESMA transação (sem collateral).

**Uso legítimo**: Arbitragem, liquidações
**Uso malicioso**: Flash loan attacks

💰 **Fee**: ~0.05-0.09% do valor

**Providers**: Aave, Uniswap v3, Balancer

🔗 *Ver Capítulo 8: Security - Flash Loan Attacks*

⚠️ **Risco**: Manipulação de preço via flash loan

*Ver também*: Arbitrage, Oracle Manipulation, DeFi

---

### **Forge**
Ferramenta de testing do Foundry.

**Comandos**:
- `forge test` - rodar testes
- `forge coverage` - coverage
- `forge snapshot` - gas benchmarks

🔗 *Ver Capítulo 6: Testing*

*Ver também*: Foundry, Anvil, Cast

---

### **Fork Testing**
Testar contra state real de blockchain (clonar mainnet localmente).

**Comando** (Foundry):
```bash
forge test --fork-url $MAINNET_RPC --fork-block-number 18000000
```

**Uso**: Testar integração com protocolos existentes (Uniswap, Aave)

🔗 *Ver Capítulo 6: Testing*

*Ver também*: Testing, Integration Test, Anvil

---

### **Foundry**
Toolkit Rust para desenvolvimento Solidity.

**Componentes**:
- **Forge**: Testing
- **Cast**: CLI para interagir com blockchain
- **Anvil**: Local node

**Vantagem**: Testes em Solidity (não JS), extremamente rápido

🔗 *Ver Capítulo 4: Ambiente de Desenvolvimento*

*Ver também*: Hardhat, Remix, Testing

---

### **Frontend**
Interface do usuário (UI) de DApp.

**Stack comum**:
- React + Ethers.js
- Wagmi hooks
- RainbowKit (wallet UI)

🔗 *Ver Capítulo 13: Frontend Integration*

*Ver também*: DApp, Ethers.js, Wallet Connection

---

### **Frontrunning**
MEV strategy: Ver transação pendente e executar antes (pagando mais gas).

**Exemplo**: User compra token → bot vê mempool → bot compra antes → preço sobe → bot vende

⚠️ **Proteção**: Private mempools (Flashbots Protect)

*Ver também*: MEV, Sandwich Attack, Mempool

---

### **Fungible**
Tokens intercambiáveis (1 token = outro token).

**Exemplo**: USDC, ETH, WBTC

**Padrão**: ERC-20

**Oposto**: Non-fungible (NFT - ERC-721)

*Ver também*: ERC-20, Token, NFT

---

### **Fuzzing**
Teste com inputs aleatórios para encontrar bugs.

**Ferramentas**:
- Foundry (built-in)
- Echidna (property-based)

**Exemplo** (Foundry):
```solidity
function testFuzz_Transfer(address to, uint256 amount) public {
    vm.assume(to != address(0));
    vm.assume(amount <= token.totalSupply());
    token.transfer(to, amount);
}
```

🔗 *Ver Capítulo 6: Testing*

*Ver também*: Testing, Echidna, Property-Based Testing

---

## G

### **Ganache**
Local blockchain para testes (Truffle Suite).

**Alternativa moderna**: Anvil (Foundry)

*Ver também*: Anvil, Local Node, Testing

---

### **Gas**
Unidade de custo computacional na EVM.

**Custo tx** = `gas used * gas price`

**Exemplo**: Transfer ERC-20 ~50k gas

💰 **Price**: Varia por demanda (1-100+ gwei)

🔗 *Ver Capítulo 7: Gas Optimization*

*Ver também*: Gas Price, Gas Limit, Gwei

---

### **Gas Limit**
Máximo de gas que transação pode consumir.

**Block gas limit**: ~30M (Ethereum)
**Tx gas limit**: Definido por sender

⚠️ **Se exceder**: Tx reverte (gas consumido é perdido)

*Ver também*: Gas, Out of Gas, Gas Optimization

---

### **Gas Price**
Preço por unidade de gas (em wei ou gwei).

**Pré-EIP-1559**: Um valor fixo
**Pós-EIP-1559**: Base fee + priority fee

💰 **1 gwei** = 10^9 wei = 0.000000001 ETH

*Ver também*: Gas, Gwei, EIP-1559

---

### **Gasless Transaction**
Transação onde usuário não paga gas (patrocinador paga).

**Métodos**:
- Meta-transactions (relayer paga)
- ERC-2612 Permit (approve via signature)
- ERC-4337 (paymaster)

**Vantagem**: Onboarding (usuário não precisa de ETH)

🔗 *Ver Capítulo 13: Frontend Integration*

*Ver também*: Meta-Transaction, Permit, Account Abstraction

---

### **Genesis Block**
Primeiro bloco da blockchain (block #0).

**Ethereum**: 30/07/2015

*Ver também*: Block, Blockchain

---

### **Governance**
Sistema de tomada de decisão (votação).

**Componentes**:
- Governance token (peso do voto)
- Proposals (sugestões)
- Quorum (participação mínima)
- Timelock (delay para executar)

**Exemplos**: Compound, Uniswap, MakerDAO

🔗 *Ver Capítulo 12: Upgradeable Contracts*

*Ver também*: DAO, Voting, Timelock

---

### **Graph (The Graph)**
Protocolo de indexing para blockchains.

**Componentes**:
- Subgraph (schema + mappings)
- GraphQL API
- Indexers (nodes que indexam)

🔗 *Ver Capítulo 14: Indexing*

*Ver também*: Subgraph, Indexing, GraphQL

---

### **GraphQL**
Linguagem de query usada pelo The Graph.

**Exemplo**:
```graphql
{
  tokens(first: 5) {
    id
    name
    symbol
  }
}
```

*Ver também*: The Graph, Subgraph, API

---

### **Gwei**
Unidade de ETH (1 gwei = 10^9 wei).

**Uso**: Medir gas price

💰 **Conversão**:
- 1 ETH = 10^9 gwei
- 1 gwei = 10^9 wei

*Ver também*: Wei, Ether, Gas Price

---

## H

### **Hack**
→ Ver **Exploit**

---

### **Halving**
Redução da block reward pela metade (Bitcoin).

**Frequência**: A cada 210k blocks (~4 anos)

**Último**: 2024 (6.25 → 3.125 BTC)

**Efeito**: Reduz inflação de BTC

*Ver também*: Block Reward, Mining, Bitcoin

---

### **Hardhat**
Framework JavaScript para desenvolvimento Ethereum.

**Recursos**:
- Testing (Mocha, Chai)
- Deployment scripts
- Hardhat Network (local node)
- Plugins ecosystem

**Alternativa**: Foundry (Rust-based)

🔗 *Ver Capítulo 4: Ambiente de Desenvolvimento*

*Ver também*: Foundry, Testing, Deployment

---

### **Hash**
Função criptográfica de one-way.

**Propriedades**:
- Determinística (mesmo input → mesmo output)
- One-way (não pode reverter)
- Collision-resistant

**Ethereum usa**: Keccak-256

**Exemplo**: `keccak256("hello")` → `0x1c8aff...`

*Ver também*: Keccak, SHA-256, Merkle Tree

---

### **Hash Rate**
Taxa de hashing total da rede (PoW).

**Unidade**: H/s (hashes per second)

**Ethereum**: Não usa mais (migrou PoS)
**Bitcoin**: ~400 EH/s (exahashes)

*Ver também*: Mining, PoW, Difficulty

---

### **Hedera**
Blockchain enterprise usando Hashgraph (não blockchain tradicional).

**Conselho**: Google, IBM, Boeing

**TPS**: ~10k

*Ver também*: Hashgraph, Enterprise, DLT

---

### **Hot Wallet**
Wallet conectada à internet (conveniente mas menos segura).

**Exemplos**: MetaMask, Coinbase Wallet

**Oposto**: Cold wallet (hardware, offline)

⚠️ **Risco**: Vulnerável a hacks se malware/phishing

*Ver também*: Wallet, Cold Wallet, Private Key

---

## I

### **IBC (Inter-Blockchain Communication)**
Protocolo cross-chain nativo do Cosmos.

**Vantagem**: Trustless (não precisa de bridge custodial)

**Uso**: Transferir tokens entre Cosmos chains

*Ver também*: Cosmos, Cross-chain, Bridge

---

### **Immutable**
Não pode ser alterado depois de criado.

**Blockchain**: Histórico imutável (não pode alterar blocos antigos)
**Smart contracts**: Código imutável (exceto com proxies)

**Solidity**:
```solidity
uint256 public immutable CREATION_TIME;
```

*Ver também*: Constant, Proxy, Upgrade

---

### **Impermanent Loss**
Perda (temporária) ao prover liquidez em AMM vs hold.

**Causa**: Preço dos tokens muda (arbitradores rebalanceiam pool)

💰 **Exemplo**: Deposita ETH + USDC → ETH sobe 2x → impermanent loss ~5.7%

**Mitigação**: Fees compensam (se volume alto)

🔗 *Ver Capítulo 10: DeFi Primitives*

*Ver também*: AMM, Liquidity Pool, LP Token

---

### **Indexed**
Parâmetro de evento que pode ser filtrado.

**Solidity**:
```solidity
event Transfer(address indexed from, address indexed to, uint256 amount);
```

**Limite**: 3 parâmetros indexed por evento

**Uso**: `filter: { from: '0x123...' }`

*Ver também*: Event, Logs, Topic

---

### **Indexing**
Processo de organizar dados on-chain para query eficiente.

**Soluções**:
- The Graph (decentralized)
- Custom (PostgreSQL + event listeners)
- Alchemy, Moralis (centralized)

🔗 *Ver Capítulo 14: Indexing*

*Ver também*: The Graph, Subgraph, Event

---

### **Inheritance**
Contract herdar funções/state de outro contract.

**Solidity**:
```solidity
contract MyToken is ERC20, Ownable {
    // Herda de ERC20 e Ownable
}
```

**Linearização**: C3 (ordem de herança importa)

*Ver também*: Abstract Contract, Interface, Virtual

---

### **Initializer**
Função que substitui constructor em proxies.

**Razão**: Constructor roda apenas no deploy (proxy precisa de init function)

**Padrão**: `initialize()` (OpenZeppelin)

**Proteção**: Garantir que roda apenas 1x

🔗 *Ver Capítulo 12: Upgradeable Contracts*

*Ver também*: Proxy, UUPS, Constructor

---

### **Interface**
Definição de funções sem implementação.

**Solidity**:
```solidity
interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
}
```

**Uso**: Interagir com contratos externos

*Ver também*: ABI, Abstract Contract, Contract Call

---

### **IPFS (InterPlanetary File System)**
Sistema de arquivos distribuído (P2P).

**Uso DeFi/NFT**: Armazenar metadata, imagens

**CID**: Identificador de conteúdo (hash do arquivo)

**Exemplo**: `ipfs://QmX...` (NFT metadata)

**Pinning services**: Pinata, Infura, web3.storage

🔗 *Ver Capítulo 15: Backend Integration*

*Ver também*: CID, Pinning, Metadata

---

## J

### **JSON-RPC**
Protocolo para comunicação com nodes Ethereum.

**Métodos comuns**:
- `eth_call` (read)
- `eth_sendTransaction` (write)
- `eth_getBalance`

**Providers**: Alchemy, Infura, QuickNode

*Ver também*: RPC, Provider, Node

---

## K

### **Keccak-256**
Função de hash usada pela Ethereum.

**Diferença SHA-3**: Keccak é versão pré-padronização

**Solidity**: `keccak256(abi.encodePacked(...))`

**Output**: 32 bytes (256 bits)

*Ver também*: Hash, Encoding, Signature

---

### **Keeper**
→ Ver **Chainlink Automation**

---

### **KYC (Know Your Customer)**
Verificação de identidade (compliance).

**CEXs**: Obrigatório
**DEXs**: Geralmente não (mas alguns protocolos DeFi exigem)

**Web2 equivalente**: Account verification

*Ver também*: CEX, Compliance, Regulation

---

## L

### **Layer 1 (L1)**
Blockchain base (mainnet).

**Exemplos**: Ethereum, Bitcoin, Solana, Cardano

**Característica**: Segurança própria (validators próprios)

*Ver também*: Layer 2, Mainnet, Blockchain

---

### **Layer 2 (L2)**
Blockchain que herda segurança de L1 (processa txs fora, settle em L1).

**Tipos**:
- **Optimistic Rollup**: Arbitrum, Optimism (fraude provável)
- **ZK Rollup**: zkSync, StarkNet (validade provada)
- **State Channel**: Lightning Network
- **Plasma**: (obsoleto)

**Vantagem**: Escalabilidade (mais TPS, menos custo)

🔗 *Ver Apêndice A: Comparativo Blockchains*

*Ver também*: Rollup, Scaling, Optimistic, ZK

---

### **LayerZero**
Protocolo de messaging cross-chain.

**Uso**: Conectar contratos em chains diferentes

**Competidores**: Axelar, Wormhole

*Ver também*: Cross-chain, Bridge, Interoperability

---

### **Lending**
Protocolo de empréstimo (borrow/lend).

**Componentes**:
- Collateral (garantia)
- Interest rate (taxa)
- Liquidation (se collateral cai muito)

**Exemplos**: Aave, Compound

🔗 *Ver Capítulo 10: DeFi Primitives*

*Ver também*: Collateral, Liquidation, Interest Rate

---

### **Library**
Contrato reutilizável (funções utilitárias).

**Solidity**:
```solidity
library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        return a + b;
    }
}
```

**Deploy**: Uma vez, usado por múltiplos contratos (economia de gas)

**OpenZeppelin**: Maior lib de utilitários

*Ver também*: OpenZeppelin, Delegatecall, Utility

---

### **Liquidation**
Venda forçada de collateral quando cai abaixo de threshold.

**Exemplo**: Empresta 500 DAI com 1000 USDC collateral → USDC cai → liquidado

💰 **Liquidation penalty**: ~5-13% (incentivo para liquidators)

**Bots**: Competem para liquidar (lucro)

🔗 *Ver Capítulo 10: DeFi - Lending*

*Ver também*: Lending, Collateral, Overcollateralized

---

### **Liquidity**
Quanto capital disponível em pool/market.

**Exemplo**: Pool Uniswap ETH/USDC com $10M TVL = alta liquidez

**Alta liquidez**: Menos slippage
**Baixa liquidez**: Mais slippage

*Ver também*: Liquidity Pool, TVL, Slippage

---

### **Liquidity Mining**
→ Ver **Yield Farming**

---

### **Liquidity Pool**
Reserva de 2+ tokens em AMM (permite trading).

**Exemplo**: Pool ETH/USDC com 100 ETH + 200k USDC

**LPs** (liquidity providers): Depositam tokens, ganham fees

**LP token**: Recibo de participação no pool

🔗 *Ver Capítulo 10: DeFi Primitives*

*Ver também*: AMM, LP Token, Impermanent Loss

---

### **Logs**
Dados emitidos por eventos (armazenados fora do state).

**Custo**: Mais barato que storage

**Uso**: Frontend escuta logs para atualizar UI

*Ver também*: Event, Indexed, Topic

---

### **LP Token**
Token que representa participação em liquidity pool.

**Exemplo**: Deposita ETH+USDC no Uniswap → recebe UNI-V2 LP token

**Composability**: Pode usar LP token em outro protocolo (staking, collateral)

*Ver também*: Liquidity Pool, AMM, ERC-20

---

## M

### **Mainnet**
Blockchain principal (produção, real money).

**Oposto**: Testnet (test money)

**Ethereum mainnet**: Chain ID 1

*Ver também*: Testnet, Chain ID, Production

---

### **Mapping**
Estrutura de dados chave-valor em Solidity.

**Exemplo**:
```solidity
mapping(address => uint256) public balances;
```

**Característica**: Não iterável (não pode listar todas as keys)

*Ver também*: Storage, State Variable, Key-Value

---

### **Merkle Tree**
Estrutura de dados para provar inclusão (usado em blockchain).

**Ethereum usa**: Transactions tree, state tree, receipts tree

**Vantagem**: Provar algo está incluído sem baixar tudo

*Ver também*: Hash, Proof, Light Client

---

### **Mempool**
Pool de transações pendentes (esperando para serem mineradas).

**MEV**: Bots analisam mempool para frontrunning

**Private mempool**: Flashbots Protect (esconde tx de bots)

*Ver também*: Transaction, MEV, Frontrunning

---

### **Meta-Transaction**
Transação assinada por usuário mas enviada (e paga) por relayer.

**Uso**: Gasless transactions (onboarding)

**Padrão**: EIP-2771

*Ver também*: Gasless, Relayer, EIP-2771

---

### **Metadata**
Dados sobre NFT (nome, descrição, imagem).

**Formato**: JSON

**Storage**: IPFS (descentralizado) ou centralized server

**Exemplo**:
```json
{
  "name": "Cool NFT #1",
  "description": "Very cool",
  "image": "ipfs://Qm..."
}
```

*Ver também*: NFT, IPFS, ERC-721

---

### **MetaMask**
Wallet browser extension mais popular.

**Concorrentes**: Coinbase Wallet, Rainbow, Rabby

**Funcionalidade**: Assinar txs, conectar com DApps

🔗 *Ver Capítulo 13: Frontend Integration*

*Ver também*: Wallet, Browser Extension, WalletConnect

---

### **MEV (Maximal Extractable Value)**
Lucro que validators/miners extraem reordenando transações.

**Tipos**:
- Frontrunning
- Backrunning
- Sandwich attacks
- Liquidations
- Arbitrage

💰 **Volume**: Bilhões de dólares por ano

**Proteção**: Flashbots Protect, private mempools

🔗 *Ver Capítulo 8: Security - Frontrunning*

*Ver também*: Frontrunning, Sandwich Attack, Mempool

---

### **Miner**
Participante que valida transações em PoW.

**Ethereum**: Não usa mais (migrou para PoS → validators)
**Bitcoin**: Ainda usa

*Ver também*: PoW, Validator, Mining

---

### **Mining**
Processo de validar transações e criar blocos (PoW).

**Recompensa**: Block reward + fees

**Ethereum**: Não usa mais (The Merge, 2022)

*Ver também*: PoW, Hash Rate, Difficulty

---

### **Mint**
Criar novos tokens.

**Exemplo**: Criar 1000 novos tokens ERC-20

**Solidity**:
```solidity
function mint(address to, uint256 amount) external onlyOwner {
    _mint(to, amount);
}
```

**Oposto**: Burn (destruir tokens)

*Ver também*: Burn, Token, Supply

---

### **Modifier**
Decorator de função em Solidity (reutilizar lógica).

**Exemplo**:
```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _;
}

function mint(uint256 amount) external onlyOwner {
    // ...
}
```

*Ver também*: Access Control, Function, Require

---

### **Moonbeam**
Parachain EVM-compatible do Polkadot.

**Uso**: Deploy código Solidity no Polkadot

*Ver também*: Polkadot, Parachain, EVM-compatible

---

### **Move**
Linguagem de smart contracts (Aptos, Sui).

**Criada por**: Meta/Facebook (para projeto Diem/Libra)

**Vantagem**: Resource safety (previne bugs)

*Ver também*: Aptos, Sui, Solidity

---

### **Multicall**
Executar múltiplas chamadas em 1 transação.

**Vantagem**: Economia de gas, atomicidade

**Solidity**:
```solidity
contract Multicall {
    function aggregate(Call[] memory calls) external returns (bytes[] memory) {
        // Executa todas as calls
    }
}
```

*Ver também*: Batch Transaction, Gas Optimization

---

### **Multi-sig**
Wallet que requer N assinaturas de M owners (ex: 3-of-5).

**Uso**: DAOs, protocolos (segurança > single key)

**Implementações**: Gnosis Safe, multisig.sol

🔗 *Ver Capítulo 18: Deployment*

⚠️ **Segurança**: Previne single point of failure

*Ver também*: Gnosis Safe, Governance, Signature

---

### **Mythril**
Security analyzer (symbolic execution) para Solidity.

**Uso**: Encontrar bugs antes de deploy

🔗 *Ver Capítulo 16: DevOps*

*Ver também*: Slither, Security, Audit

---

## N

### **NEAR**
Blockchain com sharding nativo (Rust/JS friendly).

**TPS**: 100k (teórico)
**Linguagem**: Rust, JavaScript (AssemblyScript)

🔗 *Ver Apêndice A: Comparativo Blockchains*

*Ver também*: Sharding, Rust, AssemblyScript

---

### **NFT (Non-Fungible Token)**
Token único (não intercambiável).

**Padrão**: ERC-721, ERC-1155

**Uso**: Arte digital, colecionáveis, gaming, tickets

**Marketplace**: OpenSea, Blur, LooksRare

🔗 *Ver Capítulo 9: Tokens*

*Ver também*: ERC-721, Metadata, IPFS

---

### **Nonce**
Número usado apenas uma vez.

**Account nonce**: Contador de transações (previne replay)
**Mining nonce**: Número para encontrar hash válido (PoW)

*Ver também*: Replay Attack, Transaction, Mining

---

### **Node**
Computador que roda software de blockchain (mantém cópia do ledger).

**Tipos**:
- **Full node**: Valida tudo, mantém state completo
- **Archive node**: Full node + histórico completo
- **Light node**: Apenas headers (confia em full nodes)

**Providers**: Alchemy, Infura, QuickNode (nodes-as-service)

*Ver também*: Validator, RPC, Provider

---

## O

### **Opcode**
Instrução de baixo nível que EVM executa.

**Exemplos**:
- `PUSH1`: Adicionar valor à stack
- `ADD`: Somar 2 valores
- `SSTORE`: Salvar no storage
- `CALL`: Chamar outro contrato

**Total**: ~140 opcodes

🔗 *Ver Capítulo 2: Anatomia da EVM*

*Ver também*: Bytecode, EVM, Gas

---

### **OpenZeppelin**
Biblioteca de smart contracts auditados (padrão da indústria).

**Componentes**:
- Tokens (ERC-20, ERC-721, ERC-1155)
- Access control (Ownable, AccessControl)
- Security (ReentrancyGuard, Pausable)
- Governance (Governor)
- Upgradeable (Proxies)

**Install**: `npm install @openzeppelin/contracts`

🔗 *Usado em todos os capítulos*

*Ver também*: Library, ERC, Security

---

### **Optimism**
Layer 2 Ethereum (Optimistic Rollup).

**Chain ID**: 10
**TPS**: ~2-4k
**EVM**: 100% compatível

**OP Stack**: Framework reutilizável (Base, Zora usam)

🔗 *Ver Apêndice A: Comparativo Blockchains*

*Ver também*: Optimistic Rollup, Layer 2, Base

---

### **Optimistic Rollup**
Tipo de L2 que assume transações válidas (prova apenas se desafiado).

**Vantagem**: EVM-compatible, fácil de portar código
**Desvantagem**: Withdrawal delay (~7 dias)

**Exemplos**: Arbitrum, Optimism, Base

*Ver também*: Layer 2, ZK Rollup, Fraud Proof

---

### **Oracle**
Serviço que fornece dados externos para smart contracts.

**Exemplos**:
- **Price feeds**: Chainlink, Pyth
- **Randomness**: Chainlink VRF
- **Off-chain computation**: Chainlink Functions

⚠️ **Oracle Problem**: Como confiar em dados off-chain?

🔗 *Ver Capítulo 11: Oracles*

*Ver também*: Chainlink, Oracle Problem, Price Feed

---

### **Oracle Problem**
Desafio de trazer dados confiáveis off-chain para on-chain.

**Problema**: Blockchain não pode fetch APIs (determinismo)

**Solução**: Oracles descentralizados (Chainlink)

*Ver também*: Oracle, Chainlink, Decentralization

---

### **Overcollateralized**
Collateral > valor emprestado.

**Exemplo**: Depositar $150 para emprestar $100 (150% collateralization)

**DeFi comum**: 150-200% collateralization

*Ver também*: Lending, Collateral, Liquidation

---

### **Override**
Sobrescrever função herdada.

**Solidity**:
```solidity
contract Base {
    function foo() public virtual returns (uint256) {
        return 1;
    }
}

contract Child is Base {
    function foo() public override returns (uint256) {
        return 2;
    }
}
```

*Ver também*: Virtual, Inheritance, Abstract

---

### **Ownable**
Pattern de controle de acesso (apenas owner pode executar).

**OpenZeppelin**:
```solidity
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyContract is Ownable {
    function mint() external onlyOwner {
        // Apenas owner pode mint
    }
}
```

*Ver também*: Access Control, Modifier, OpenZeppelin

---

## P

### **Parachain**
Blockchain que se conecta à Relay Chain do Polkadot (shared security).

**Slot**: Alugado via auction (lock DOT)

**Exemplos**: Moonbeam, Acala, Astar

*Ver também*: Polkadot, Relay Chain, Shared Security

---

### **Payable**
Keyword Solidity que permite função receber ETH.

```solidity
function deposit() external payable {
    // Pode enviar ETH junto
}
```

**msg.value**: Quantidade de ETH enviada

*Ver também*: msg.value, Ether, Transfer

---

### **Paymaster**
Contrato que paga gas por usuário (ERC-4337).

**Uso**: Gasless transactions, sponsored gas

*Ver também*: Account Abstraction, Gasless, ERC-4337

---

### **Permit**
Approve via assinatura (sem gastar gas).

**Padrão**: ERC-2612

**Uso**: Gasless approve

**Exemplo**: DAI, USDC (versão 2)

*Ver também*: EIP-712, Gasless, Signature

---

### **Phishing**
Ataque de engenharia social (roubar private keys).

**Exemplos**:
- Site falso (metamask-phishing.com)
- Discord DM (fake airdrop)
- Malicious approve (drainer contract)

⚠️ **Proteção**: Sempre verificar URL, nunca compartilhar seed phrase

*Ver também*: Security, Scam, Drainer

---

### **Polkadot**
Blockchain de blockchains (parachains).

**Linguagem**: Rust (Substrate, Ink!)
**Consensus**: NPoS (Nominated Proof of Stake)

🔗 *Ver Apêndice A: Comparativo Blockchains*

*Ver também*: Parachain, Substrate, Relay Chain

---

### **Pool**
→ Ver **Liquidity Pool**, **Staking Pool**

---

### **PoS (Proof of Stake)**
Consensus onde validators fazem stake de tokens.

**Ethereum**: Usa PoS (pós-Merge)
**Stake**: 32 ETH para validator

**Vantagem**: Mais eficiente energeticamente que PoW

*Ver também*: Validator, Staking, Slashing

---

### **PoW (Proof of Work)**
Consensus onde miners competem resolvendo puzzle criptográfico.

**Bitcoin**: Usa PoW
**Ethereum**: Usava (migrou para PoS em 2022)

**Desvantagem**: Alto consumo energético

*Ver também*: Mining, Hash Rate, Difficulty

---

### **Price Feed**
Oracle que fornece preço de asset.

**Chainlink**: Mais usado (ETH/USD, BTC/USD, etc)

**Update**: A cada X segundos ou Y% de mudança

🔗 *Ver Capítulo 11: Oracles*

*Ver também*: Oracle, Chainlink, Pyth

---

### **Priority Fee**
Tip para validator (pós-EIP-1559).

💰 **Total fee** = base fee (queimada) + priority fee (para validator)

*Ver também*: Gas, EIP-1559, Base Fee

---

### **Private Key**
Chave secreta que controla account/wallet.

**Formato**: 64 hex chars (32 bytes)

⚠️ **NUNCA compartilhe**: Quem tem private key controla fundos

**Derivação**: Private key → Public key → Address

*Ver também*: Public Key, Seed Phrase, EOA

---

### **Program (Solana)**
Equivalente ao smart contract na Solana.

**Linguagem**: Rust (Anchor framework)

*Ver também*: Solana, Smart Contract, Anchor

---

### **Proof**
Evidência criptográfica de algo.

**Tipos**:
- Merkle proof (inclusão em tree)
- ZK proof (sabe algo sem revelar)
- Fraud proof (transação inválida em Optimistic Rollup)

*Ver também*: Merkle Tree, ZK, Fraud Proof

---

### **Protocol**
Conjunto de smart contracts que implementam funcionalidade (DEX, lending, etc).

**Exemplos**: Uniswap (DEX), Aave (lending), MakerDAO (stablecoin)

*Ver também*: DeFi, Smart Contract, DApp

---

### **Provider**
Serviço que fornece acesso a node via RPC.

**Exemplos**: Alchemy, Infura, QuickNode, Ankr

**Ethers.js**:
```javascript
const provider = new ethers.JsonRpcProvider("https://eth.llamarpc.com");
```

*Ver também*: RPC, Node, JSON-RPC

---

### **Proxy Pattern**
Separar lógica (implementation) de storage (proxy) para permitir upgrades.

**Tipos**:
- **Transparent Proxy** (OpenZeppelin)
- **UUPS** (implementação tem upgrade logic)
- **Diamond** (múltiplas implementations)

🔗 *Ver Capítulo 12: Upgradeable Contracts*

*Ver também*: Upgradeable, UUPS, Diamond, Delegatecall

---

### **Public Key**
Chave pública derivada de private key.

**Uso**: Verificar assinaturas

**Ethereum**: Não é o address (address = last 20 bytes de keccak256(pubkey))

*Ver também*: Private Key, Address, ECDSA

---

### **Pull Over Push**
Design pattern: Permitir que usuários retirem (pull) ao invés de enviar automaticamente (push).

**Razão**: Segurança (evitar reentrancy, falha em transfer)

**Exemplo**:
```solidity
// ❌ Push (perigoso)
function distribute() external {
    payable(user1).transfer(amount1);
    payable(user2).transfer(amount2); // Falha aqui = toda tx reverte
}

// ✅ Pull (seguro)
function withdraw() external {
    uint256 amount = pendingWithdrawals[msg.sender];
    pendingWithdrawals[msg.sender] = 0;
    payable(msg.sender).transfer(amount);
}
```

🔗 *Ver Capítulo 5: Design Patterns*

*Ver também*: Design Pattern, Reentrancy, Withdraw Pattern

---

### **Pure**
Função que não lê nem modifica state.

**Solidity**:
```solidity
function add(uint256 a, uint256 b) public pure returns (uint256) {
    return a + b;
}
```

*Ver também*: View, Function, State

---

### **Pyth Network**
Oracle de preços (competitor do Chainlink).

**Diferença**: On-demand updates (Chainlink é push-based)

**Chains**: Multi-chain (Solana, EVM, Aptos)

*Ver também*: Oracle, Price Feed, Chainlink

---

## Q

### **Quorum**
Número mínimo de votos necessários para proposta passar.

**Exemplo**: DAO requer 40% dos tokens votarem para passar proposal

*Ver também*: DAO, Governance, Voting

---

## R

### **Randomness**
Número aleatório (difícil de gerar on-chain).

⚠️ **Problema**: `block.timestamp`, `blockhash` são previsíveis (validators podem manipular)

**Solução**: Chainlink VRF (Verifiable Random Function)

🔗 *Ver Capítulo 11: Oracles - VRF*

*Ver também*: VRF, Chainlink, Oracle

---

### **Receive**
Função executada quando contrato recebe ETH (sem calldata).

**Solidity**:
```solidity
receive() external payable {
    // Recebe ETH direto (sem função)
}
```

*Ver também*: Fallback, Payable, Transfer

---

### **Reentrancy**
Vulnerabilidade onde função é chamada novamente antes de terminar.

**Ataque clássico**: DAO hack (2016, $60M)

**Proteção**: ReentrancyGuard, Checks-Effects-Interactions

**Exemplo vulnerável**:
```solidity
function withdraw() external {
    uint256 amount = balances[msg.sender];
    (bool success, ) = msg.sender.call{value: amount}("");
    balances[msg.sender] = 0; // ❌ Tarde demais (reentrancy possível)
}
```

🔗 *Ver Capítulo 8: Security - Reentrancy*

⚠️ **Crítico**: Atualizar state ANTES de external calls

*Ver também*: CEI Pattern, ReentrancyGuard, Security

---

### **ReentrancyGuard**
Modifier OpenZeppelin que previne reentrancy.

**Uso**:
```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MyContract is ReentrancyGuard {
    function withdraw() external nonReentrant {
        // Protegido contra reentrancy
    }
}
```

*Ver também*: Reentrancy, Security, Modifier

---

### **Relay Chain**
Blockchain central do Polkadot (conecta parachains).

**Função**: Segurança compartilhada, consensus

*Ver também*: Polkadot, Parachain, Shared Security

---

### **Relayer**
Serviço que envia transação em nome de usuário (meta-transactions).

**Uso**: Gasless transactions (relayer paga gas)

*Ver também*: Meta-Transaction, Gasless, ERC-2771

---

### **Remix**
IDE online para Solidity.

**URL**: remix.ethereum.org

**Recursos**: Editor, compiler, debugger, deploy

**Vantagem**: Zero setup (browser-based)

🔗 *Ver Capítulo 4: Ambiente de Desenvolvimento*

*Ver também*: IDE, Foundry, Hardhat

---

### **Reorganization (Reorg)**
Mudança de canonical chain (block é "desfeito").

**Causa**: Dois miners encontram block simultaneamente → chain fork → um vence

**Profundidade**: Ethereum ~1-2 blocks, Bitcoin raro > 1

**Proteção**: Esperar múltiplas confirmações

*Ver também*: Finality, Confirmation, Fork

---

### **Replay Attack**
Reusar transação assinada em contexto diferente.

**Proteções**:
- Nonce (previne replay na mesma chain)
- Chain ID (previne replay entre chains)
- Domain separator (EIP-712)

*Ver também*: Signature, Nonce, Chain ID

---

### **Require**
Validação em Solidity (reverte se falso).

**Exemplo**:
```solidity
require(msg.sender == owner, "Not owner");
```

**Gas**: Retorna gas restante (vs `assert` que consome tudo)

*Ver também*: Assert, Revert, Error

---

### **Revert**
Reverter transação (desfazer mudanças de state).

**Solidity**:
```solidity
if (amount > balance) {
    revert InsufficientBalance(amount, balance);
}
```

**Custom errors** (0.8.4+): Mais eficiente que string

🔗 *Ver Capítulo 7: Gas Optimization*

*Ver também*: Require, Custom Error, Transaction

---

### **Rollup**
Tipo de L2 que "rola" múltiplas transações em 1.

**Tipos**:
- **Optimistic**: Assume válido (Arbitrum, Optimism)
- **ZK**: Prova criptográfica (zkSync, StarkNet)

*Ver também*: Layer 2, Optimistic Rollup, ZK Rollup

---

### **RPC (Remote Procedure Call)**
Protocolo para comunicação com blockchain node.

**Endpoint exemplo**: `https://eth.llamarpc.com`

**Providers**: Alchemy, Infura, QuickNode

*Ver também*: Provider, Node, JSON-RPC

---

### **Royalty**
Percentual pago ao criador em cada venda secundária (NFT).

**Padrão**: ERC-2981

**Problema**: Não enforce on-chain (marketplace pode ignorar)

*Ver também*: NFT, ERC-2981, Marketplace

---

## S

### **Sandwich Attack**
MEV strategy: Frontrun + backrun vítima.

**Exemplo**:
1. Vítima compra token (pending mempool)
2. Bot frontrun: Compra antes (preço sobe)
3. Tx vítima executa (preço alto)
4. Bot backrun: Vende (lucro)

⚠️ **Proteção**: Slippage tolerance, private mempool

🔗 *Ver Capítulo 8: Security - MEV*

*Ver também*: MEV, Frontrunning, Slippage

---

### **Scam**
→ Ver **Phishing**, **Rug Pull**

---

### **Seed Phrase**
12-24 palavras que geram private keys (BIP-39).

**Exemplo**: "witch collapse practice feed shame open despair creek road again ice least"

⚠️ **NUNCA compartilhe**: Recupera toda wallet

**Derivação**: Seed → Master key → Account keys

*Ver também*: Private Key, Wallet, BIP-39

---

### **Selfdestruct**
Opcode que destrói contrato e envia ETH para address.

⚠️ **Deprecated**: Será removido em futuro (não usar)

**Problema**: Pode forçar ETH em contratos (security issue)

*Ver também*: Opcode, Deprecation

---

### **Sepolia**
Testnet Ethereum (substituiu Goerli/Ropsten).

**Faucet**: Pega ETH gratuito para testes

**Chain ID**: 11155111

*Ver também*: Testnet, Goerli, Faucet

---

### **Shared Security**
Múltiplas chains usam mesmos validators (segurança compartilhada).

**Exemplo**: Polkadot parachains herdam segurança da Relay Chain

**Vantagem**: Chains novas não precisam de validators próprios

*Ver também*: Polkadot, Parachain, Validator

---

### **Signer**
Account que assina transações.

**Ethers.js**:
```javascript
const signer = new ethers.Wallet(privateKey, provider);
await contract.connect(signer).transfer(to, amount);
```

*Ver também*: Wallet, Private Key, Provider

---

### **Signature**
Prova criptográfica de que mensagem foi assinada por private key.

**Algoritmo**: ECDSA (secp256k1 no Ethereum)

**Componentes**: r, s, v (recovery ID)

**Uso**: Autenticação, Permit, Meta-transactions

*Ver também*: ECDSA, EIP-712, Private Key

---

### **Slashing**
Penalidade para validator malicioso (perda de stake).

**Ethereum PoS**: Validator pode perder 1-100% do stake

**Razões**: Dupla assinatura, inatividade prolongada

*Ver também*: PoS, Validator, Staking

---

### **Slippage**
Diferença entre preço esperado e preço executado.

**Causa**: Liquidez baixa ou trade grande

**Exemplo**: Espera comprar por $100, mas executa por $102 (2% slippage)

**Proteção**: `amountOutMin` (reverte se slippage > tolerância)

🔗 *Ver Capítulo 10: DeFi - AMM*

*Ver também*: AMM, Liquidity, Price Impact

---

### **Slither**
Static analyzer para Solidity (Trail of Bits).

**Uso**: Encontrar vulnerabilidades antes de deploy

**Detectors**: 70+ (reentrancy, overflow, etc)

🔗 *Ver Capítulo 16: DevOps*

*Ver também*: Mythril, Security, Audit

---

### **Smart Contract**
Código auto-executável na blockchain.

**Ethereum**: Escrito em Solidity/Vyper, compilado para bytecode

**Immutável**: Não pode alterar depois de deploy (exceto proxies)

🔗 *Ver Capítulo 3: Solidity*

*Ver também*: Solidity, EVM, DApp

---

### **Solana**
Blockchain de alta performance (PoH + PoS).

**TPS**: 3000-5000 (testnet: 65k)
**Custo**: ~$0.00025/tx
**Linguagem**: Rust (Anchor)

🔗 *Ver Apêndice A: Comparativo Blockchains*

*Ver também*: Rust, Anchor, SVM

---

### **Solhint**
Linter para Solidity.

**Uso**: Enforce code style, best practices

**Config**: `.solhint.json`

*Ver também*: Linting, Code Quality, CI/CD

---

### **Solidity**
Linguagem principal para smart contracts Ethereum.

**Versão atual**: 0.8.x

**Syntax**: Similar a JavaScript/TypeScript

🔗 *Ver Capítulo 3: Solidity*

*Ver também*: Vyper, Smart Contract, EVM

---

### **Stablecoin**
Crypto com preço estável (~$1).

**Tipos**:
- **Fiat-backed**: USDC, USDT (collateral em banco)
- **Crypto-backed**: DAI (overcollateralized com ETH)
- **Algorithmic**: UST (falhou), FRAX

*Ver também*: USDC, DAI, Collateral

---

### **Staking**
Lock tokens para ganhar rewards.

**Tipos**:
- **PoS validation**: Validar blockchain (Ethereum: 32 ETH)
- **DeFi staking**: Depositar em protocolo (yield farming)

💰 **Rewards**: 3-20% APY (varia)

🔗 *Ver Capítulo 10: DeFi - Staking*

*Ver também*: PoS, Validator, Yield

---

### **State**
Dados atuais da blockchain (balances, storage, etc).

**Ethereum**: State = conjunto de todos os accounts e storage

**State root**: Hash da Merkle tree do state (em cada block)

*Ver também*: Storage, Merkle Tree, Block

---

### **Storage**
Persistência de dados on-chain (caro).

**Custo**: ~20k gas para criar slot, 5k para atualizar

**Layout**: 256-bit slots

🔗 *Ver Capítulo 7: Gas Optimization - Storage Packing*

*Ver também*: Memory, Calldata, State

---

### **Storage Collision**
Bug em proxies onde storage do proxy sobrescreve storage da implementation.

**Prevenção**: Padrão de storage gap, namespaced storage (EIP-1967)

🔗 *Ver Capítulo 12: Upgradeable Contracts*

*Ver também*: Proxy, Delegatecall, Storage

---

### **Subgraph**
Definição de como indexar dados blockchain (The Graph).

**Componentes**:
- Schema (GraphQL)
- Mappings (AssemblyScript)
- Manifest (config)

🔗 *Ver Capítulo 14: Indexing*

*Ver também*: The Graph, Indexing, GraphQL

---

### **SVM (Solana Virtual Machine)**
Virtual machine da Solana (diferente de EVM).

**Características**: Parallel execution, accounts model

*Ver também*: EVM, Solana, Virtual Machine

---

## T

### **Tenderly**
Plataforma de monitoring e debugging para smart contracts.

**Recursos**:
- Transaction simulator
- Debugger
- Alerts
- Web3 Actions

🔗 *Ver Capítulo 19: Monitoring*

*Ver também*: Monitoring, Debugging, Alerts

---

### **Testnet**
Blockchain de testes (ETH/tokens sem valor).

**Ethereum testnets**:
- Sepolia (atual)
- Goerli (descontinuado)
- Holešky (validator testing)

**Faucets**: Pega ETH grátis

*Ver também*: Mainnet, Sepolia, Faucet

---

### **The Graph**
→ Ver **Graph (The Graph)**

---

### **The Merge**
Migração do Ethereum de PoW para PoS (15/09/2022).

**Mudanças**:
- Mining → Staking
- 99.95% redução energética
- Issuance reduzida (~90%)

*Ver também*: PoS, PoW, Validator

---

### **Timelock**
Delay obrigatório antes de executar função admin.

**Uso**: DAOs, protocolos (segurança - comunidade tem tempo para reagir)

**Delay comum**: 24h-7 dias

🔗 *Ver Capítulo 12: Upgradeable Contracts*

*Ver também*: DAO, Governance, Multi-sig

---

### **Token**
Asset digital na blockchain.

**Tipos**:
- **Fungible**: ERC-20 (USDC, UNI)
- **Non-fungible**: ERC-721 (NFTs)
- **Semi-fungible**: ERC-1155

🔗 *Ver Capítulo 9: Tokens*

*Ver também*: ERC-20, ERC-721, Cryptocurrency

---

### **Token ID**
Identificador único de NFT.

**Exemplo**: Bored Ape #1234 (token ID = 1234)

*Ver também*: NFT, ERC-721, Metadata

---

### **Topic**
Campo indexado de event (permite filtrar logs).

**Limite**: 4 topics por event (topic0 = event signature, outros 3 = indexed params)

**Uso**: `filter: { topics: [eventSignature, from, to] }`

*Ver também*: Event, Indexed, Logs

---

### **Total Supply**
Quantidade total de tokens existentes.

**Exemplo**: 21 milhões BTC (fixo), ETH ~120M (inflacionário)

**Solidity**:
```solidity
function totalSupply() public view returns (uint256) {
    return _totalSupply;
}
```

*Ver também*: Mint, Burn, Circulating Supply

---

### **Transaction**
Operação assinada que muda state da blockchain.

**Componentes**:
- From (sender)
- To (receiver ou contract)
- Value (ETH enviado)
- Data (calldata)
- Gas limit
- Gas price
- Nonce
- Signature (v, r, s)

*Ver também*: Gas, Nonce, Signature

---

### **Transfer**
Enviar ETH ou tokens.

**ETH transfer**:
```solidity
payable(to).transfer(amount); // ❌ Deprecated (2300 gas limit)
(bool success, ) = to.call{value: amount}(""); // ✅ Recomendado
```

**ERC-20 transfer**:
```solidity
token.transfer(to, amount);
```

*Ver também*: Call, Send, ERC-20

---

### **Transparent Proxy**
Proxy pattern onde admin não pode chamar implementation.

**Razão**: Prevenir função selector collision

**OpenZeppelin**: Padrão usado

🔗 *Ver Capítulo 12: Upgradeable Contracts*

*Ver também*: Proxy, UUPS, Delegatecall

---

### **TVL (Total Value Locked)**
Valor total depositado em protocolo DeFi.

💰 **Métrica de sucesso**: Quanto maior TVL, mais confiança

**Exemplo**: Uniswap ~$4B TVL, Aave ~$6B

**Tracking**: DeFiLlama

*Ver também*: DeFi, Liquidity, Protocol

---

### **Tx**
→ Abreviação de **Transaction**

---

## U

### **Unchecked**
Bloco Solidity sem overflow/underflow checks (economia de gas).

**Solidity 0.8+**:
```solidity
unchecked {
    counter++; // Sem overflow check (economiza ~30 gas)
}
```

⚠️ **Usar com cuidado**: Apenas quando garantir que não overflow

🔗 *Ver Capítulo 7: Gas Optimization*

*Ver também*: Overflow, Gas Optimization, SafeMath

---

### **Uniswap**
DEX AMM mais popular (constant product).

**Versões**:
- v2: AMM básico
- v3: Concentrated liquidity
- v4: Hooks (customização)

**Formula**: `x * y = k`

🔗 *Ver Capítulo 10: DeFi Primitives*

*Ver também*: AMM, DEX, Liquidity Pool

---

### **Upgradeable**
Contrato que pode ter lógica alterada (via proxy).

**Padrões**: Transparent Proxy, UUPS, Diamond

⚠️ **Risco**: Centralization (admin pode mudar lógica)

🔗 *Ver Capítulo 12: Upgradeable Contracts*

*Ver também*: Proxy, Immutable, Governance

---

### **UUPS (Universal Upgradeable Proxy Standard)**
Proxy onde lógica de upgrade está na implementation (não no proxy).

**Vantagem**: Mais eficiente em gas que Transparent Proxy

**OpenZeppelin**: `UUPSUpgradeable`

🔗 *Ver Capítulo 12: Upgradeable Contracts*

*Ver também*: Proxy, Transparent Proxy, Upgradeable

---

### **USDC**
Stablecoin centralizada ($1, emitida por Circle).

**Supply**: ~$25B
**Backing**: Dólar em banco
**Decimals**: 6 (não 18!)

*Ver também*: Stablecoin, USDT, DAI

---

### **USDT (Tether)**
Stablecoin mais usada (~$1).

**Supply**: ~$90B (maior stablecoin)
**Blockchain**: Ethereum, Tron, outros

*Ver também*: Stablecoin, USDC, Tron

---

## V

### **Validator**
Participante que valida transações em PoS.

**Ethereum**: Precisa de 32 ETH staked

**Rewards**: Block rewards + fees
**Penalidades**: Slashing (se malicioso)

*Ver também*: PoS, Staking, Slashing

---

### **Vault**
Contrato que guarda assets e implementa estratégia (yield).

**Padrão**: ERC-4626

**Exemplos**: Yearn vaults, Beefy

*Ver também*: ERC-4626, Yield Farming, DeFi

---

### **View**
Função que lê state mas não modifica.

**Solidity**:
```solidity
function balanceOf(address account) public view returns (uint256) {
    return balances[account];
}
```

**Gas**: Grátis (quando chamada externally)

*Ver também*: Pure, Function, State

---

### **Virtual**
Função que pode ser sobrescrita por contrato filho.

**Solidity**:
```solidity
function foo() public virtual returns (uint256) {
    return 1;
}
```

*Ver também*: Override, Inheritance, Abstract

---

### **VRF (Verifiable Random Function)**
Randomness verificável (Chainlink VRF).

**Uso**: Lotteries, NFT minting, gaming

**Vantagem**: Provável que é realmente aleatório (não manipulável)

🔗 *Ver Capítulo 11: Oracles*

*Ver também*: Randomness, Chainlink, Oracle

---

### **Vyper**
Linguagem alternativa ao Solidity (Python-like).

**Vantagens**: Sintaxe Python, menos footguns
**Desvantagens**: Ecosystem menor

**Exemplo**:
```python
@external
def transfer(to: address, amount: uint256):
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
```

*Ver também*: Solidity, Python, Smart Contract

---

## W

### **Wallet**
Software que gerencia private keys e assina transações.

**Tipos**:
- **Hot**: Online (MetaMask, Coinbase Wallet)
- **Cold**: Offline (Ledger, Trezor)
- **Smart**: Contract-based (Gnosis Safe, Argent)

*Ver também*: MetaMask, Private Key, Seed Phrase

---

### **WalletConnect**
Protocolo para conectar wallets mobile com DApps.

**Uso**: Scan QR code → conecta wallet

**Vantagem**: Não precisa de browser extension

*Ver também*: Wallet, MetaMask, DApp

---

### **WASM (WebAssembly)**
Bytecode format usado em algumas blockchains.

**Chains**: NEAR, Polkadot (Ink!), CosmWasm

**Vantagem**: Múltiplas linguagens podem compilar para WASM (Rust, C, Go)

*Ver também*: Bytecode, EVM, Rust

---

### **Web3**
Termo para internet descentralizada (blockchain-based).

**Web1**: Read (static sites)
**Web2**: Read + Write (Facebook, Google)
**Web3**: Read + Write + **Own** (blockchain, self-custody)

*Ver também*: Blockchain, Decentralization, DApp

---

### **Web3.js**
Biblioteca JavaScript para Ethereum (predecessor do Ethers.js).

**Status**: Ainda mantida mas Ethers.js mais popular

*Ver também*: Ethers.js, JavaScript, Provider

---

### **Wei**
Menor unidade de ETH (1 ETH = 10^18 wei).

**Conversão**:
- 1 wei = 0.000000000000000001 ETH
- 1 gwei = 10^9 wei
- 1 ETH = 10^18 wei

**Solidity**: Valores são em wei por padrão

*Ver também*: Gwei, Ether, Decimals

---

### **Whitelist**
Lista de addresses permitidos (access control).

**Uso**: Presale NFT, early access

**Solidity**:
```solidity
mapping(address => bool) public whitelist;

modifier onlyWhitelisted() {
    require(whitelist[msg.sender], "Not whitelisted");
    _;
}
```

*Ver também*: Access Control, Presale, Modifier

---

### **Withdraw Pattern**
→ Ver **Pull Over Push**

---

### **Wrapped Token**
Representação de asset de outra chain.

**Exemplos**:
- WETH (Wrapped ETH - ERC-20 version de ETH)
- WBTC (Bitcoin em Ethereum)
- stETH (Staked ETH - Lido)

**Uso**: Permitir ETH em contratos ERC-20

*Ver também*: Bridge, WETH, Cross-chain

---

## Y

### **Yield**
Retorno/lucro de investment.

**DeFi**: Staking, liquidity providing, lending

💰 **Medido em**: APY (Annual Percentage Yield)

*Ver também*: APY, Yield Farming, Staking

---

### **Yield Farming**
Estratégia de mover capital entre protocolos para maximizar yield.

**Exemplo**: Depositar USDC no Aave → emprestar DAI → stakear DAI em Curve

**Risco**: Impermanent loss, smart contract risk, rug pulls

*Ver também*: Yield, DeFi, Liquidity Mining

---

## Z

### **Zero Address**
Address 0x0000000000000000000000000000000000000000.

**Usos**:
- Burn tokens (enviar para zero address)
- Check `require(to != address(0))` (prevenir erros)

⚠️ **Importante**: Nunca enviar assets para zero address (perdidos para sempre)

*Ver também*: Burn, Address, Validation

---

### **ZK (Zero-Knowledge)**
Provar algo sem revelar informação.

**Exemplo**: Provar idade > 18 sem revelar idade exata

**Blockchain**: ZK Rollups (provar transações válidas sem enviar todas)

**Tipos**:
- **zk-SNARK**: Ethereum, zkSync, StarkNet
- **zk-STARK**: StarkNet (sem trusted setup)

*Ver também*: ZK Rollup, Privacy, Proof

---

### **ZK Rollup**
L2 que usa zero-knowledge proofs para validação.

**Vantagem**: Finalidade rápida (proof matemático)
**Desvantagem**: Complexidade, EVM compatibility difícil

**Exemplos**: zkSync Era, Polygon zkEVM, StarkNet

*Ver também*: Layer 2, Rollup, Optimistic Rollup

---

### **zkEVM**
EVM-compatible ZK Rollup.

**Desafio**: ZK circuits são difíceis de implementar para EVM

**Exemplos**: Polygon zkEVM, zkSync Era, Scroll

*Ver também*: ZK Rollup, EVM, Layer 2

---

### **zkSync**
ZK Rollup Ethereum.

**Versões**:
- zkSync Lite (payments)
- zkSync Era (EVM-compatible)

**Chain ID**: 324 (Era mainnet)

*Ver também*: ZK Rollup, Layer 2, zkEVM

---

## Conclusão

**Você agora tem um glossário completo de 300+ termos Web3!**

**Como usar este apêndice:**
1. 🔍 **Referência rápida**: Ctrl+F para encontrar termos
2. 📚 **Estudo**: Ler categoria por categoria (A-Z)
3. 🔗 **Deep dive**: Seguir links para capítulos do ebook
4. 💡 **Comparações**: Entender Web3 via analogias Web2

**Termos mais importantes para dominar primeiro:**
1. **Smart Contract**, **EVM**, **Gas**
2. **EOA**, **Private Key**, **Wallet**
3. **ERC-20**, **ERC-721**, **Token**
4. **DeFi**, **AMM**, **DEX**
5. **Reentrancy**, **Overflow**, **Delegatecall** (security)
6. **Proxy**, **Upgradeable**, **UUPS**
7. **Oracle**, **Chainlink**, **The Graph**
8. **Layer 2**, **Rollup**, **Optimistic**

**Próximo apêndice**: Recursos e comunidades (onde aprender, quem seguir, como se manter atualizado). 🌐
