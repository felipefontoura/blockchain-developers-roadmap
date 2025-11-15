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
