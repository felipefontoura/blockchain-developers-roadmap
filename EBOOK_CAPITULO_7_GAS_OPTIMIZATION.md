# Capítulo 7: Gas Optimization - Por Que e Como

> **Para Desenvolvedores Experientes**: Se você otimiza loops em Python ou queries em SQL, vai entender gas optimization. Mas aqui, otimizar não é apenas velocidade - é literalmente economizar dinheiro dos usuários. Cada operação tem custo real em USD.

---

## Índice
- [7.1 Por Que Otimizar Gas](#71-por-que-otimizar-gas)
- [7.2 Storage Packing](#72-storage-packing)
- [7.3 Memory vs Calldata](#73-memory-vs-calldata)
- [7.4 Short-Circuiting](#74-short-circuiting)
- [7.5 Unchecked Blocks](#75-unchecked-blocks)
- [7.6 Custom Errors](#76-custom-errors)
- [7.7 Loops e Batch Operations](#77-loops-e-batch-operations)
- [7.8 Assembly (Yul)](#78-assembly-yul)

---

## 7.1 Por Que Otimizar Gas

### Custo Real

```
Exemplo real (Ethereum mainnet, gas = 50 gwei):

Operação            | Gas   | Custo (ETH @ $2000)
--------------------|-------|--------------------
Transfer ERC-20     | 65k   | $6.50
Mint NFT            | 150k  | $15.00
Swap Uniswap        | 180k  | $18.00
Deploy contrato     | 2M    | $200.00
```

**Otimizar 20% = economizar $1.30 por transfer!**

Com 1000 users/dia = **$1,300/dia economizados**.

---

## 7.2 Storage Packing

### Storage Slots (32 bytes cada)

```solidity
// ❌ INEFICIENTE: 5 slots (100k gas para setar todos)
uint256 a;  // Slot 0 (32 bytes)
uint8 b;    // Slot 1 (1 byte, desperdiça 31!)
uint256 c;  // Slot 2
uint8 d;    // Slot 3 (desperdiça 31 bytes!)
uint256 e;  // Slot 4

// ✅ OTIMIZADO: 3 slots (60k gas - economiza 40k!)
uint256 a;  // Slot 0
uint256 c;  // Slot 1
uint256 e;  // Slot 2
uint8 b;    // Slot 3 (1 byte)
uint8 d;    // Slot 3 (1 byte) - PACKED!
```

### Packing Inteligente

```solidity
struct User {
    // ❌ MAU: 4 slots
    uint256 id;          // 32 bytes
    address wallet;      // 20 bytes
    bool active;         // 1 byte
    uint256 balance;     // 32 bytes

    // ✅ BOM: 3 slots
    uint256 id;          // Slot 0
    uint256 balance;     // Slot 1
    address wallet;      // Slot 2 (20 bytes)
    uint96 extra;        // Slot 2 (12 bytes) - packed!
    bool active;         // Slot 2 (1 byte) - packed!
}
```

**Regra**: Agrupe variáveis pequenas para caber em 32 bytes.

---

## 7.3 Memory vs Calldata

```solidity
// ❌ CARO: copia para memory
function processData(uint256[] memory data) public {
    // Copia todo array para memory = caro!
    uint256 first = data[0]; // ~400 gas
}

// ✅ BARATO: usa calldata diretamente
function processData(uint256[] calldata data) external {
    // Lê direto do calldata = barato!
    uint256 first = data[0]; // ~200 gas
}
```

**Regra**: `external` + `calldata` para arrays/strings.

---

## 7.4 Short-Circuiting

```solidity
// ❌ Ordem ruim
function check() public view returns (bool) {
    return expensiveCheck() && cheapCheck();
    // Sempre executa expensiveCheck primeiro
}

// ✅ Ordem boa
function check() public view returns (bool) {
    return cheapCheck() && expensiveCheck();
    // Se cheapCheck falha, não executa expensiveCheck
}

// ✅ Ainda melhor: cache
bool public cached;

function check() public view returns (bool) {
    if (cached) return true; // Return early
    return expensiveCheck();
}
```

---

## 7.5 Unchecked Blocks

```solidity
// Solidity 0.8+: checks overflow automático = custa gas

// ❌ Com checks (padrão)
function loop() public {
    for (uint256 i = 0; i < 1000; i++) {
        // i++ tem overflow check (~20 gas)
    }
}

// ✅ Sem checks (SE VOCÊ TEM CERTEZA)
function loop() public {
    for (uint256 i = 0; i < 1000; ) {
        unchecked { i++; } // Sem check (~3 gas)
    }
    // Economiza ~17 gas × 1000 = 17k gas!
}
```

⚠️ **WARNING**: Só use se TEM CERTEZA que não vai overflow!

---

## 7.6 Custom Errors

```solidity
// ❌ CARO: string errors
require(balance >= amount, "Insufficient balance"); // ~50 bytes

// ✅ BARATO: custom errors (Solidity 0.8.4+)
error InsufficientBalance(uint256 available, uint256 required);

if (balance < amount) {
    revert InsufficientBalance(balance, amount); // ~22 bytes
}
```

**Economia**: ~50% menos gas + informação tipada!

---

## 7.7 Loops e Batch Operations

```solidity
// ❌ Loop em storage (MUITO CARO)
uint256[] public items;

function badUpdate() public {
    for (uint i = 0; i < items.length; i++) {
        items[i] = i * 2; // SSTORE cada iteração!
    }
}

// ✅ Cache length
function betterUpdate() public {
    uint256 length = items.length; // 1 SLOAD
    for (uint i = 0; i < length; i++) {
        items[i] = i * 2;
    }
}

// ✅✅ Batch em memory
function bestUpdate() public {
    uint256[] memory temp = items; // Copia uma vez
    for (uint i = 0; i < temp.length; i++) {
        temp[i] = i * 2; // Memory (barato)
    }
    // Depois atualiza storage se necessário
}
```

---

## 7.8 Assembly (Yul)

### Quando Usar

Otimizações extremas (< 1% dos casos).

```solidity
// Normal: ~500 gas
function normalAdd(uint a, uint b) public pure returns (uint) {
    return a + b;
}

// Assembly: ~450 gas
function assemblyAdd(uint a, uint b) public pure returns (uint result) {
    assembly {
        result := add(a, b)
    }
}
```

### Exemplo Prático: Custom Storage

```solidity
contract AssemblyExample {
    // Storage slot específico (evita collisions)
    bytes32 constant SLOT = keccak256("my.custom.slot");

    function getValue() public view returns (uint256 value) {
        assembly {
            value := sload(SLOT)
        }
    }

    function setValue(uint256 value) public {
        assembly {
            sstore(SLOT, value)
        }
    }
}
```

⚠️ **Cuidado**: Assembly é perigoso! Só use se REALMENTE precisa.

---

## 📖 Glossário

**SLOAD**
> Opcode para ler storage. Custo: 2,100 gas (cold) / 100 gas (warm).

**SSTORE**
> Opcode para escrever storage. Custo: 20,000 gas (novo) / 5,000 gas (update).

**Cold vs Warm**
> Primeiro acesso a slot = cold (caro). Acessos seguintes = warm (barato).

**Yul**
> Linguagem assembly intermediária do Solidity.

---

## 🔒 Checklist de Otimização

### Sempre Faça
- [ ] Storage packing (agrupe variáveis pequenas)
- [ ] `calldata` para external functions com arrays
- [ ] Custom errors em vez de strings
- [ ] Cache `.length` em loops

### Considere Fazer
- [ ] `unchecked` em loops (SE seguro)
- [ ] Short-circuit em condições
- [ ] Batch operations
- [ ] `immutable`/`constant` quando possível

### Raramente Faça
- [ ] Assembly (apenas se extremamente necessário)
- [ ] Inline assembly para operações críticas

---

## 📝 Exercício: Gas Golf

Otimize este contrato:

```solidity
contract Unoptimized {
    uint256 public count;
    address public owner;
    bool public active;
    uint256[] public items;

    function update(uint256[] memory newItems) public {
        require(msg.sender == owner, "Not authorized");
        require(active == true, "Contract not active");

        for (uint256 i = 0; i < newItems.length; i++) {
            items.push(newItems[i]);
        }

        count = count + 1;
    }
}
```

**Meta**: Reduzir gas em 30%+

<details>
<summary>✅ Solução</summary>

```solidity
// Otimizações aplicadas:
// 1. Storage packing
// 2. Calldata
// 3. Custom error
// 4. Unchecked
// 5. Cache

contract Optimized {
    // Storage packing: owner + active cabem em 1 slot
    address public owner;      // 20 bytes
    bool public active;        // 1 byte - PACKED!
    uint256 public count;
    uint256[] public items;

    // Custom error
    error NotAuthorized();
    error NotActive();

    function update(uint256[] calldata newItems) external {
        if (msg.sender != owner) revert NotAuthorized();
        if (!active) revert NotActive();

        uint256 length = newItems.length;
        for (uint256 i; i < length; ) {
            items.push(newItems[i]);
            unchecked { ++i; }
        }

        unchecked { ++count; }
    }
}
```

**Economia**: ~40% menos gas!
</details>

---

## 📚 Recursos

1. **[Gas Optimization Tips](https://github.com/Dravee/gas-optimization-tips)**
2. **[Solidity Gas Optimizations](https://gist.github.com/hrkrshnn/ee8fabd532058307229d65dcd5836ddc)**

---

**Próximo**: Capítulo 9 - Tokens (aplicar otimizações em ERC-20/721).
