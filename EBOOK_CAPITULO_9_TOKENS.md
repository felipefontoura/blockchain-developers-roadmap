# Capítulo 9: Tokens - ERC-20, ERC-721, ERC-1155

> **Para Desenvolvedores Experientes**: Tokens são building blocks de blockchain. ERC-20 é como JSON para APIs - padrão universal. ERC-721 (NFTs) é unique IDs on steroids. ERC-1155 é otimização dos dois. Domine estes e você entende 80% de DeFi.

---

## Índice
- [9.1 ERC-20 - Fungible Tokens](#91-erc-20---fungible-tokens)
- [9.2 ERC-721 - NFTs](#92-erc-721---nfts)
- [9.3 ERC-1155 - Multi-Token](#93-erc-1155---multi-token)
- [9.4 IPFS para Metadata](#94-ipfs-para-metadata)
- [9.5 Token Economics Básico](#95-token-economics-básico)

---

## 9.1 ERC-20 - Fungible Tokens

### Interface Padrão

```solidity
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

### Implementação Completa

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ERC20 {
    string public name;
    string public symbol;
    uint8 public decimals = 18;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }

    function transfer(address to, uint256 amount) public returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) public returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) public returns (bool) {
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }

    function mint(address to, uint256 amount) public {
        balanceOf[to] += amount;
        totalSupply += amount;
        emit Transfer(address(0), to, amount);
    }

    function burn(address from, uint256 amount) public {
        balanceOf[from] -= amount;
        totalSupply -= amount;
        emit Transfer(from, address(0), amount);
    }
}
```

### Approve/TransferFrom Pattern

```solidity
// User aprova DEX
token.approve(dexAddress, 1000);

// DEX pode transferir em nome do user
token.transferFrom(userAddress, dexAddress, 1000);
```

**Uso Real**: DEXs, staking, lending.

---

## 🔍 Estudo de Caso: USDC e USDT - ERC-20 em Produção

### USDC - Centre Consortium

**Contrato**: USDC (USD Coin)
**Endereço Ethereum**: [0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48](https://etherscan.io/token/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48)
**Market Cap**: ~$30B+
**Uso**: Stablecoin mais usada em DeFi

### Implementação Tutorial vs Produção

**Nossa Implementação Tutorial**:
```solidity
// Simples, sem controles
contract ERC20 {
    function transfer(address to, uint256 amount) public returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
}
```

**USDC em Produção**:
```solidity
// USDC é upgradeable (usa proxy pattern)
contract FiatTokenV2_2 is FiatTokenV2_1 {
    // Blacklist capability
    mapping(address => bool) internal blacklisted;

    modifier notBlacklisted(address account) {
        require(!blacklisted[account], "Blacklistable: account is blacklisted");
        _;
    }

    // Pausable
    bool public paused = false;

    modifier whenNotPaused() {
        require(!paused, "Pausable: paused");
        _;
    }

    function transfer(address to, uint256 value)
        external
        override
        whenNotPaused
        notBlacklisted(msg.sender)
        notBlacklisted(to)
        returns (bool)
    {
        _transfer(msg.sender, to, value);
        return true;
    }

    // Admin functions
    function pause() external onlyPauser {
        paused = true;
        emit Pause();
    }

    function blacklist(address account) external onlyBlacklister {
        blacklisted[account] = true;
        emit Blacklisted(account);
    }
}
```

### Diferenças-Chave: Tutorial → Produção

| Recurso | Tutorial | USDC (Produção) | Por Quê |
|---------|----------|-----------------|---------|
| **Upgradeable** | ❌ Não | ✅ Proxy Pattern | Corrigir bugs, adicionar features |
| **Pausable** | ❌ Não | ✅ Emergency stop | Congelar em caso de exploit |
| **Blacklist** | ❌ Não | ✅ Compliance/OFAC | Regulação, endereços sancionados |
| **Role-based** | ❌ Simples owner | ✅ Multiple roles | Minter, Pauser, Blacklister separados |
| **Gas Optimization** | ❌ Básico | ✅ Otimizado | Bilhões em volume, cada gas conta |

### USDT - Tether (Caso Especial ⚠️)

**Contrato**: USDT (Tether)
**Endereço**: [0xdAC17F958D2ee523a2206206994597C13D831ec7](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7)
**Market Cap**: ~$120B+

**⚠️ USDT NÃO segue ERC-20 perfeitamente!**

```solidity
// ERC-20 padrão: transfer retorna bool
function transfer(address to, uint256 value) external returns (bool);

// USDT: NÃO retorna nada!
function transfer(address to, uint value) public;
```

**Problema**: Código que espera `bool` pode quebrar!

```solidity
// ❌ Pode falhar com USDT!
bool success = usdt.transfer(recipient, amount);
require(success, "Transfer failed");

// ✅ Solução: SafeERC20 (OpenZeppelin)
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

using SafeERC20 for IERC20;
usdt.safeTransfer(recipient, amount); // Funciona!
```

### Lições para Produção

1. **Sempre use SafeERC20**: Lida com tokens não-padrão (USDT)
2. **Considere pausable**: Botão de emergência é crítico
3. **Upgradeable é comum**: Stablecoins quase sempre são
4. **Compliance existe**: Blacklist é realidade em stablecoins reguladas
5. **Teste com tokens reais**: USDC/USDT/DAI, não só mocks

---

## 9.2 ERC-721 - NFTs

### Interface

```solidity
interface IERC721 {
    function balanceOf(address owner) external view returns (uint256);
    function ownerOf(uint256 tokenId) external view returns (address);
    function transferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;
    function getApproved(uint256 tokenId) external view returns (address);
    function setApprovalForAll(address operator, bool approved) external;
    function isApprovedForAll(address owner, address operator) external view returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
}
```

### Implementação

```solidity
contract ERC721 {
    string public name;
    string public symbol;

    mapping(uint256 => address) public ownerOf;
    mapping(address => uint256) public balanceOf;
    mapping(uint256 => address) public getApproved;
    mapping(address => mapping(address => bool)) public isApprovedForAll;

    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }

    function transferFrom(address from, address to, uint256 tokenId) public {
        require(from == ownerOf[tokenId], "Not owner");
        require(
            msg.sender == from ||
            msg.sender == getApproved[tokenId] ||
            isApprovedForAll[from][msg.sender],
            "Not authorized"
        );

        balanceOf[from]--;
        balanceOf[to]++;
        ownerOf[tokenId] = to;
        delete getApproved[tokenId];

        emit Transfer(from, to, tokenId);
    }

    function approve(address to, uint256 tokenId) public {
        address owner = ownerOf[tokenId];
        require(msg.sender == owner || isApprovedForAll[owner][msg.sender]);

        getApproved[tokenId] = to;
        emit Approval(owner, to, tokenId);
    }

    function setApprovalForAll(address operator, bool approved) public {
        isApprovedForAll[msg.sender][operator] = approved;
        emit ApprovalForAll(msg.sender, operator, approved);
    }

    function mint(address to, uint256 tokenId) public {
        require(ownerOf[tokenId] == address(0), "Already minted");

        balanceOf[to]++;
        ownerOf[tokenId] = to;

        emit Transfer(address(0), to, tokenId);
    }

    function burn(uint256 tokenId) public {
        address owner = ownerOf[tokenId];
        require(msg.sender == owner);

        balanceOf[owner]--;
        delete ownerOf[tokenId];

        emit Transfer(owner, address(0), tokenId);
    }

    // ERC721Metadata
    function tokenURI(uint256 tokenId) public view returns (string memory) {
        require(ownerOf[tokenId] != address(0), "Token doesn't exist");
        return string(abi.encodePacked("ipfs://", tokenId));
    }
}
```

---

## 🔍 Estudo de Caso: ERC721A - NFTs Otimizados para Produção

### Azuki NFT Collection

**Contrato**: Azuki
**Endereço**: [0xED5AF388653567Af2F388E6224dC7C4b3241C544](https://etherscan.io/address/0xed5af388653567af2f388e6224dc7c4b3241c544)
**Supply**: 10,000 NFTs
**Volume**: $700M+ (all-time)
**Inovação**: ERC721A - batch minting otimizado

### O Problema com ERC-721 Padrão

**Mint 5 NFTs tradicional**:
```solidity
// ❌ ERC-721 básico: 5 transações separadas
for(uint i = 0; i < 5; i++) {
    mint(msg.sender, tokenId++); // ~140k gas CADA = 700k total
}
```

**Custo**: ~140k gas × 5 = **700k gas**

**Por quê é caro?**
- Cada `mint()` escreve no storage:
  - `ownerOf[tokenId] = owner` (20k gas)
  - `balanceOf[owner]++` (20k gas)
  - Emit event (5k gas)

### Solução: ERC721A

**Criado por**: Chiru Labs (equipe Azuki)
**Otimização**: Batch minting com storage otimizado

```solidity
// ✅ ERC721A: Mint 5 NFTs em 1 transação
contract ERC721A {
    // Struct para ownership + metadata
    struct TokenOwnership {
        address addr;
        uint64 startTimestamp;
    }

    // Só grava owner do PRIMEIRO token do batch
    mapping(uint256 => TokenOwnership) internal _ownerships;
    mapping(address => uint256) internal _balances;

    uint256 internal _currentIndex = 0;

    function _safeMint(address to, uint256 quantity) internal {
        uint256 startTokenId = _currentIndex;

        // Apenas 1 write no storage para o batch inteiro!
        _ownerships[startTokenId] = TokenOwnership(to, uint64(block.timestamp));

        _balances[to] += quantity;
        _currentIndex += quantity;

        // Emit events para cada token
        for (uint256 i = 0; i < quantity; i++) {
            emit Transfer(address(0), to, startTokenId + i);
        }
    }

    function ownerOf(uint256 tokenId) public view returns (address) {
        require(tokenId < _currentIndex, "Token doesn't exist");

        // Procura "para trás" até achar owner gravado
        for (uint256 curr = tokenId; ; curr--) {
            TokenOwnership memory ownership = _ownerships[curr];
            if (ownership.addr != address(0)) {
                return ownership.addr;
            }
        }
    }
}
```

### Como Funciona

**Mint batch de 5 NFTs (IDs #100-104)**:
```
Storage write:
_ownerships[100] = Alice  // Só grava o primeiro!
_balances[Alice] += 5     // 1 write

Tokens #101-104 não têm entry!
```

**Ao consultar `ownerOf(103)`**:
```
1. _ownerships[103] = 0x0 (vazio)
2. _ownerships[102] = 0x0 (vazio)
3. _ownerships[101] = 0x0 (vazio)
4. _ownerships[100] = Alice ✓ (encontrou!)
   → Retorna Alice
```

### Comparação de Gas

| Operação | ERC-721 | ERC721A | Economia |
|----------|---------|---------|----------|
| Mint 1 NFT | ~140k gas | ~140k gas | 0% |
| Mint 5 NFTs | ~700k gas | ~180k gas | **74%** 💰 |
| Mint 10 NFTs | ~1.4M gas | ~220k gas | **84%** 💰 |
| Transfer | ~50k gas | ~50k gas | 0% |
| First transfer após batch mint | ~50k gas | ~70k gas | -40% ⚠️ |

### Trade-offs do ERC721A

**✅ Vantagens**:
- **Mint em batch muito mais barato**: Essencial para collections de 10k
- **Compatível com ERC-721**: Marketplaces funcionam normalmente
- **Timestamp embutido**: Saber quando foi mintado

**⚠️ Desvantagens**:
- **Primeira transferência mais cara**: Precisa "preencher" ownership
- **ownerOf() loop**: Mais gas em read (não afeta transações)
- **Complexidade**: Mais difícil de auditar

### Quando Usar ERC721A?

**✅ Use se**:
- NFT collection com mints em batch (5+ por vez)
- Public mint onde users mintam múltiplos
- Precisa minimizar gas de mint

**❌ Não use se**:
- Mint sempre 1 por vez
- Transfers mais frequentes que mints
- Precisa código simples para auditoria

### Código Real da Azuki

```solidity
// Simplified from actual Azuki contract
contract Azuki is ERC721A, Ownable {
    uint256 public constant MAX_SUPPLY = 10000;
    uint256 public constant MAX_PER_TX = 5;

    function mint(uint256 quantity) external payable {
        require(totalSupply() + quantity <= MAX_SUPPLY, "Sold out");
        require(quantity <= MAX_PER_TX, "Max 5 per tx");
        require(msg.value >= 1 ether * quantity, "Not enough ETH");

        _safeMint(msg.sender, quantity); // ERC721A magic!
    }
}
```

### Lições para Produção

1. **Otimize mint, não transfer**: Users mintam 1x, mas podem transferir 100x
2. **Considere trade-offs**: Gas savings vs complexidade
3. **ERC721A é padrão para 10k collections**: Usado por BAYC, Doodles, etc.
4. **Sempre teste batch edge cases**: Batch de 1, de 100, etc.
5. **Documentação crítica**: Código complexo precisa comentários

### Alternativas Modernas

**ERC721Psi** (PSI = Permanent Storage Increase):
- Similar ao ERC721A
- Melhor para transferências frequentes

**ERC721Consecutive** (EIP-2309):
- Padrão oficial para batch minting
- Menos otimizado que ERC721A

---

## 9.3 ERC-1155 - Multi-Token

### Por Que ERC-1155?

**Problema**: Jogo tem 1000 tipos de items → 1000 contratos ERC-721?
**Solução**: 1 contrato ERC-1155 com 1000 IDs!

```solidity
interface IERC1155 {
    function balanceOf(address account, uint256 id) external view returns (uint256);
    function balanceOfBatch(address[] calldata accounts, uint256[] calldata ids) external view returns (uint256[] memory);
    function setApprovalForAll(address operator, bool approved) external;
    function isApprovedForAll(address account, address operator) external view returns (bool);
    function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes calldata data) external;
    function safeBatchTransferFrom(address from, address to, uint256[] calldata ids, uint256[] calldata amounts, bytes calldata data) external;

    event TransferSingle(address indexed operator, address indexed from, address indexed to, uint256 id, uint256 value);
    event TransferBatch(address indexed operator, address indexed from, address indexed to, uint256[] ids, uint256[] values);
}
```

### Implementação Simplificada

```solidity
contract ERC1155 {
    mapping(uint256 => mapping(address => uint256)) public balanceOf;
    mapping(address => mapping(address => bool)) public isApprovedForAll;

    event TransferSingle(address indexed operator, address indexed from, address indexed to, uint256 id, uint256 value);

    function safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes calldata data
    ) public {
        require(msg.sender == from || isApprovedForAll[from][msg.sender]);

        balanceOf[id][from] -= amount;
        balanceOf[id][to] += amount;

        emit TransferSingle(msg.sender, from, to, id, amount);
    }

    function mint(address to, uint256 id, uint256 amount) public {
        balanceOf[id][to] += amount;
        emit TransferSingle(msg.sender, address(0), to, id, amount);
    }
}
```

---

## 9.4 IPFS para Metadata

### Problema

NFT image não vai on-chain (muito caro!).

### Solução: IPFS

```solidity
contract NFT is ERC721 {
    string public baseURI = "ipfs://QmYourCIDHere/";

    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        return string(abi.encodePacked(baseURI, tokenId.toString(), ".json"));
    }
}
```

**Metadata JSON** (em IPFS):
```json
{
  "name": "My NFT #1",
  "description": "Cool NFT",
  "image": "ipfs://QmImageCID/1.png",
  "attributes": [
    {"trait_type": "Rarity", "value": "Legendary"},
    {"trait_type": "Power", "value": 100}
  ]
}
```

---

## 9.5 Token Economics Básico

### Supply Models

```solidity
// Fixed Supply
contract FixedSupply is ERC20 {
    constructor() {
        _mint(msg.sender, 1_000_000 * 10**18); // 1M tokens
    }
    // Sem mint, sem burn = supply fixo
}

// Inflationary
contract Inflationary is ERC20 {
    address public minter;

    function mint(address to, uint256 amount) public {
        require(msg.sender == minter);
        _mint(to, amount); // Supply cresce
    }
}

// Deflationary
contract Deflationary is ERC20 {
    function transfer(address to, uint256 amount) public override returns (bool) {
        uint256 burnAmount = amount / 100; // 1% burn
        _burn(msg.sender, burnAmount);
        return super.transfer(to, amount - burnAmount);
    }
}
```

---

## 📖 Glossário

**Fungible**
> Tokens intercambiáveis (1 ETH = 1 ETH). ERC-20.

**Non-Fungible**
> Tokens únicos (NFT #1 ≠ NFT #2). ERC-721.

**Allowance**
> Permissão para terceiro gastar tokens em seu nome.

**Metadata**
> Dados descritivos (nome, imagem, atributos) de NFT.

**IPFS**
> InterPlanetary File System - storage descentralizado.

---

## 🔒 Security Checklist

- [ ] Reentrancy guard em funções que transferem
- [ ] Overflow checks (0.8+ automático)
- [ ] Approve race condition (use increaseAllowance)
- [ ] Mint/burn access control
- [ ] TokenURI validates tokenId exists

---

## 📝 Exercício

Implemente token ERC-20 com:
- Fixed supply
- Pausable
- Burnable
- Role-based minting (apenas MINTER_ROLE)

---

## 📚 Recursos

1. **[OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)**
2. **[ERC-20 Standard](https://eips.ethereum.org/EIPS/eip-20)**
3. **[ERC-721 Standard](https://eips.ethereum.org/EIPS/eip-721)**

---

**Próximo**: Capítulo 10 - DeFi Primitives (usar tokens em DEXs, lending).

---

**Última Atualização**: 2025-01-17
**Changelog**: Adicionados estudos de caso de contratos reais (USDC, USDT, ERC721A/Azuki)
