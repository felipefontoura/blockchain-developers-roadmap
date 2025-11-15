# Capítulo 3: Solidity - A Linguagem e Suas Peculiaridades

> **Para Desenvolvedores Experientes**: Se você conhece TypeScript, C++ ou Java, a sintaxe de Solidity será familiar. Mas cuidado - as similaridades superficiais escondem diferenças profundas. Esta linguagem foi projetada para rodar em uma máquina virtual bizantina, em milhares de computadores simultaneamente, onde cada operação custa dinheiro. Isso muda tudo.

---

## Índice
- [3.1 Solidity vs Linguagens Tradicionais](#31-solidity-vs-linguagens-tradicionais)
- [3.2 Type System - O que é Diferente](#32-type-system---o-que-é-diferente)
- [3.3 Value Types vs Reference Types](#33-value-types-vs-reference-types)
- [3.4 Storage, Memory e Calldata - Na Prática](#34-storage-memory-e-calldata---na-prática)
- [3.5 Funções - Visibility e Modificadores](#35-funções---visibility-e-modificadores)
- [3.6 Structs, Arrays e Mappings](#36-structs-arrays-e-mappings)
- [3.7 Herança e Interfaces](#37-herança-e-interfaces)
- [3.8 Peculiaridades Críticas](#38-peculiaridades-críticas)

---

## 3.1 Solidity vs Linguagens Tradicionais

### Comparação Rápida

| Aspecto | JavaScript/TypeScript | Solidity |
|---------|----------------------|----------|
| **Tipagem** | Dinâmica (JS) / Estática (TS) | Estática, forte |
| **Integers** | Number (64-bit float) | uint8 até uint256 (precision exata) |
| **Overflow** | Infinity ou wraparound | Revert (desde 0.8.0) |
| **Null/Undefined** | null, undefined | Valores default (0, false, "") |
| **Strings** | UTF-16, mutáveis | UTF-8, imutáveis por padrão |
| **Arrays** | Dinâmicos, métodos built-in | Fixos ou dinâmicos, menos métodos |
| **Objects** | key-value flexível | Structs com types fixos |
| **Maps** | Map, Object | mapping (sem iteração!) |
| **This** | Contexto de execução | address(this) = contrato |
| **Classes** | ES6 classes | Contracts (similar) |
| **Async** | Promises, async/await | Síncrono (mas external calls) |
| **Errors** | try/catch | require/revert/assert |
| **Custo** | CPU time | Gas ($ real) |

### Exemplo Lado-a-Lado

**TypeScript**:
```typescript
class Token {
    private balances: Map<string, number> = new Map();

    transfer(from: string, to: string, amount: number): boolean {
        const senderBalance = this.balances.get(from) || 0;

        if (senderBalance < amount) {
            throw new Error("Insufficient balance");
        }

        this.balances.set(from, senderBalance - amount);
        this.balances.set(to, (this.balances.get(to) || 0) + amount);

        return true;
    }
}
```

**Solidity**:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Token {
    mapping(address => uint256) private balances;

    function transfer(address to, uint256 amount) public returns (bool) {
        uint256 senderBalance = balances[msg.sender]; // default = 0

        // Revert em vez de throw
        require(senderBalance >= amount, "Insufficient balance");

        // Overflow checked automaticamente (0.8+)
        balances[msg.sender] = senderBalance - amount;
        balances[to] = balances[to] + amount;

        return true;
    }
}
```

**Diferenças críticas**:
1. `msg.sender` é global (quem chamou a função)
2. `mapping` não pode ser iterado
3. `require` reverte TODA transação (não apenas retorna erro)
4. Gas é consumido por cada operação
5. Tudo é público na blockchain (mesmo `private`!)

---

## 3.2 Type System - O que é Diferente

### 📖 Tipos Básicos

**Integers com Precisão Específica**

```solidity
// Unsigned (apenas positivos)
uint8   // 0 to 255
uint16  // 0 to 65,535
uint32  // 0 to 4,294,967,295
uint64  // 0 to 18,446,744,073,709,551,615
uint128 // 0 to 340,282,366,920,938,463,463,374,607,431,768,211,455
uint256 // 0 to 2^256 - 1 (padrão, use "uint" como alias)

// Signed (positivos e negativos)
int8    // -128 to 127
int16   // -32,768 to 32,767
...
int256  // -2^255 to 2^255 - 1 (use "int" como alias)
```

**Por que tantos tamanhos?**
> Gas optimization! Variáveis menores podem ser "packed" no storage.

```solidity
// ❌ INEFICIENTE - cada variável usa 1 slot (32 bytes)
uint256 a = 10;  // Slot 0
uint256 b = 20;  // Slot 1
uint256 c = 30;  // Slot 2
// Total: 3 slots

// ✅ OTIMIZADO - todas cabem em 1 slot
uint128 a = 10;  // Slot 0 (16 bytes)
uint128 b = 20;  // Slot 0 (16 bytes)
uint256 c = 30;  // Slot 1
// Total: 2 slots (economiza 1 SSTORE = ~15k gas!)
```

**Address Types**

```solidity
address      // 20 bytes (160 bits) - 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb
address payable // Pode receber ETH (.transfer, .send)

// Conversão
address owner = 0x123...; // read-only
address payable recipient = payable(owner); // pode receber ETH
```

**Boolean**

```solidity
bool public isActive = true;
bool public isPaused; // default = false

// Operadores lógicos (como outras linguagens)
if (isActive && !isPaused) {
    // ...
}
```

**Bytes**

```solidity
// Tamanho fixo
bytes1  b1 = 0xff;        // 1 byte
bytes8  b8 = 0x123456;    // 8 bytes
bytes32 b32 = keccak256(abi.encodePacked("hello")); // 32 bytes (common para hashes)

// Tamanho dinâmico
bytes   data = hex"001122"; // Array de bytes
string  text = "Hello";     // Equivalente a bytes, mas UTF-8

// ⚠️ bytes32 vs bytes
// bytes32 = mais barato (value type, cabe em 1 slot)
// bytes = mais caro (reference type, storage aponta para data)
```

**Enums**

```solidity
enum Status {
    Pending,    // 0
    Active,     // 1
    Completed,  // 2
    Cancelled   // 3
}

Status public currentStatus = Status.Pending;

function activate() public {
    currentStatus = Status.Active;
}

// Enums economizam gas vs uint256 (uint8 é suficiente)
```

---

## 3.3 Value Types vs Reference Types

### Diferença Crítica

**Value Types**: Copiados quando passados
- `uint`, `int`, `bool`, `address`, `bytes1-32`, `enum`

**Reference Types**: Passados por referência
- `arrays`, `struct`, `mapping`, `string`, `bytes`

### Comportamento de Cópia

**Value Types** (sempre copiados):
```solidity
function example() public {
    uint256 a = 10;
    uint256 b = a;  // CÓPIA de valor
    b = 20;         // 'a' continua 10

    assert(a == 10);  // ✅ true
    assert(b == 20);  // ✅ true
}
```

**Reference Types** (comportamento depende de storage/memory):
```solidity
contract ReferenceExample {
    uint[] public storageArray = [1, 2, 3];

    function modifyArray() public {
        // Referência ao storage (MODIFICA original)
        uint[] storage ref = storageArray;
        ref[0] = 999;

        assert(storageArray[0] == 999); // ✅ modificou original
    }

    function copyArray() public {
        // Cópia para memory (NÃO modifica original)
        uint[] memory copy = storageArray;
        copy[0] = 999;

        assert(storageArray[0] == 1);   // ✅ original não mudou
        assert(copy[0] == 999);          // ✅ cópia mudou
    }
}
```

---

## 3.4 Storage, Memory e Calldata - Na Prática

### Quando Usar Cada Um

| Data Location | Usa em | Persist | Mutável | Custo Gas |
|---------------|--------|---------|---------|-----------|
| **storage** | State variables | ✅ Sim | ✅ Sim | 💰💰💰 Caro |
| **memory** | Função params/local vars | ❌ Não | ✅ Sim | 💰 Barato |
| **calldata** | External function params | ❌ Não | ❌ Não | 💰 Mais barato |

### Regras de Usage

```solidity
contract DataLocations {
    // State variables = sempre storage (implícito)
    uint256[] public storageArray;

    // ✅ Parâmetros de external function = preferir calldata
    function processExternal(uint256[] calldata data) external {
        // calldata = read-only, mais barato que memory
        uint256 first = data[0]; // OK
        // data[0] = 999; // ❌ ERRO: calldata é imutável
    }

    // ✅ Parâmetros de public function = memory
    function processPublic(uint256[] memory data) public {
        // memory = mutável, mas não persiste
        data[0] = 999; // OK (mas não afeta caller)
    }

    // ✅ Retornar arrays = memory
    function getArray() public view returns (uint256[] memory) {
        return storageArray; // Copia storage → memory
    }

    // ✅ Modificar state = storage reference
    function modifyState() public {
        uint256[] storage ref = storageArray;
        ref.push(999); // Modifica storageArray
    }
}
```

### Pegadinha Comum: Storage Pointer

```solidity
contract StoragePointerBug {
    struct User {
        string name;
        uint256 balance;
    }

    User[] public users;

    // ❌ BUG: cria storage pointer vazio por acidente
    function badFunction() public {
        User storage user; // Aponta para storage slot 0!
        user.balance = 100; // Sobrescreve variável não relacionada!
    }

    // ✅ CORRETO: sempre inicialize storage pointers
    function goodFunction() public {
        users.push(User("Alice", 0));
        User storage user = users[0]; // Referência válida
        user.balance = 100;
    }
}
```

⚠️ **Warning**: Compilador moderno (0.8+) previne alguns desses bugs, mas cuidado em código legado!

---

## 3.5 Funções - Visibility e Modificadores

### Function Visibility

```solidity
contract VisibilityExample {
    uint256 private counter;

    // public: qualquer um pode chamar, gera getter automático se state var
    function publicFunction() public returns (uint256) {
        return counter;
    }

    // external: apenas chamadas externas (mais eficiente que public)
    function externalFunction() external returns (uint256) {
        return counter;
    }

    // internal: apenas este contrato e derivados (herança)
    function internalFunction() internal returns (uint256) {
        return counter;
    }

    // private: apenas este contrato
    function privateFunction() private returns (uint256) {
        return counter;
    }
}
```

**Comparação de Gas**:
```solidity
// external = mais barato (calldata)
function processExternal(uint256[] calldata data) external {
    // ~200 gas para ler data[0]
}

// public = mais caro (copia para memory)
function processPublic(uint256[] memory data) public {
    // ~400 gas para ler data[0]
}

// Se função só é chamada externamente, use external!
```

### State Mutability

```solidity
contract Mutability {
    uint256 public value;

    // view: lê state, não modifica
    function getValue() public view returns (uint256) {
        return value; // ✅ OK ler
        // value = 10; // ❌ ERRO não pode modificar
    }

    // pure: não lê nem modifica state
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b; // ✅ OK
        // return value; // ❌ ERRO não pode ler state
    }

    // payable: pode receber ETH
    function deposit() public payable {
        value += msg.value; // ETH recebido
    }

    // (default): pode modificar state, não recebe ETH
    function setValue(uint256 newValue) public {
        value = newValue;
    }
}
```

💡 **Pro Tip**: `view` e `pure` não custam gas quando chamadas externamente (via node RPC), apenas quando chamadas por outra transação!

### Custom Modifiers

```solidity
contract ModifierExample {
    address public owner;
    bool public paused;

    constructor() {
        owner = msg.sender;
    }

    // Modifier = reutilizar lógica de validação
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _; // Continue execução da função
    }

    modifier whenNotPaused() {
        require(!paused, "Paused");
        _;
    }

    // Usar múltiplos modifiers
    function criticalAction() public onlyOwner whenNotPaused {
        // Só executa se msg.sender == owner E !paused
        // ...
    }

    // Modifier com parâmetros
    modifier costs(uint256 price) {
        require(msg.value >= price, "Insufficient payment");
        _;
    }

    function buyItem() public payable costs(1 ether) {
        // Só executa se msg.value >= 1 ETH
        // ...
    }
}
```

---

## 3.6 Structs, Arrays e Mappings

### Structs

```solidity
struct User {
    address wallet;
    string name;
    uint256 balance;
    bool active;
}

// Usar em storage
mapping(address => User) public users;

function createUser(string memory name) public {
    // Método 1: Constructor style
    users[msg.sender] = User({
        wallet: msg.sender,
        name: name,
        balance: 0,
        active: true
    });

    // Método 2: Positional
    users[msg.sender] = User(msg.sender, name, 0, true);

    // Método 3: Modificar campos individualmente
    User storage user = users[msg.sender];
    user.wallet = msg.sender;
    user.name = name;
    user.balance = 0;
    user.active = true;
}
```

### Arrays

**Arrays Fixos vs Dinâmicos**:
```solidity
// Fixo (tamanho definido em compile-time)
uint256[5] public fixedArray;

// Dinâmico (pode crescer)
uint256[] public dynamicArray;

// Métodos
function arrayMethods() public {
    // Push (apenas dinâmicos)
    dynamicArray.push(10);  // Adiciona ao final

    // Pop (remove último)
    dynamicArray.pop();     // Remove e retorna último

    // Length
    uint256 len = dynamicArray.length;

    // Delete (set para default, não remove)
    delete dynamicArray[0]; // Vira 0, mas length não muda!

    // Criar em memory
    uint256[] memory tempArray = new uint256[](5); // Tamanho fixo em memory
}
```

⚠️ **Pegadinha**: `delete` em array não remove elemento, apenas zera!

```solidity
// ❌ ERRADO: tentar "remover" com delete
uint256[] public numbers = [1, 2, 3];
delete numbers[1]; // numbers = [1, 0, 3] - length ainda é 3!

// ✅ CORRETO: remover realmente (shift elementos)
function removeAt(uint256 index) public {
    require(index < numbers.length);

    for (uint256 i = index; i < numbers.length - 1; i++) {
        numbers[i] = numbers[i + 1];
    }
    numbers.pop();
}

// ✅ ALTERNATIVA: swap com último e pop (mais eficiente, mas não mantém ordem)
function removeSwap(uint256 index) public {
    require(index < numbers.length);
    numbers[index] = numbers[numbers.length - 1];
    numbers.pop();
}
```

### Mappings

**A Estrutura de Dados Mais Usada em Solidity**

```solidity
// Sintaxe: mapping(KeyType => ValueType)
mapping(address => uint256) public balances;

// Nested mappings
mapping(address => mapping(address => uint256)) public allowances;

// Mapping de structs
mapping(address => User) public users;

// ⚠️ LIMITAÇÕES CRÍTICAS:
// 1. Não pode iterar sobre keys!
mapping(address => uint256) public balances;
// ❌ for (address user in balances) {} // NÃO EXISTE!

// 2. Não tem .length
// ❌ balances.length // NÃO EXISTE!

// 3. Não pode deletar mapping inteiro
// ❌ delete balances; // NÃO FUNCIONA!

// 4. Sempre retorna valor default se key não existe
function checkBalance(address user) public view returns (uint256) {
    return balances[user]; // Retorna 0 se nunca foi setado
}
```

**Como Iterar Sobre Mapping** (workaround):
```solidity
contract IterableMapping {
    mapping(address => uint256) public balances;
    address[] public userAddresses; // Track keys manualmente

    function addUser(address user, uint256 balance) public {
        if (balances[user] == 0) {
            // Novo usuário
            userAddresses.push(user);
        }
        balances[user] = balance;
    }

    function getAllBalances() public view returns (address[] memory, uint256[] memory) {
        uint256[] memory amounts = new uint256[](userAddresses.length);

        for (uint256 i = 0; i < userAddresses.length; i++) {
            amounts[i] = balances[userAddresses[i]];
        }

        return (userAddresses, amounts);
    }
}
```

💡 **Pro Tip**: Para casos complexos, considere usar library [EnumerableMap do OpenZeppelin](https://docs.openzeppelin.com/contracts/4.x/api/utils#EnumerableMap).

---

## 3.7 Herança e Interfaces

### Herança (Inheritance)

```solidity
// Contrato base
contract Ownable {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    function transferOwnership(address newOwner) public onlyOwner {
        owner = newOwner;
    }
}

// Herança simples
contract MyToken is Ownable {
    uint256 public totalSupply;

    // Pode usar onlyOwner porque herda de Ownable
    function mint(uint256 amount) public onlyOwner {
        totalSupply += amount;
    }
}

// Herança múltipla
contract ERC20 {
    mapping(address => uint256) public balances;
}

contract Pausable {
    bool public paused;

    modifier whenNotPaused() {
        require(!paused, "Paused");
        _;
    }
}

// Ordem importa! (C3 linearization)
contract MyAdvancedToken is ERC20, Pausable, Ownable {
    // Herda de todos
}
```

**Ordem de Herança e `super`**:
```solidity
contract A {
    function foo() public virtual returns (string memory) {
        return "A";
    }
}

contract B is A {
    function foo() public virtual override returns (string memory) {
        return "B";
    }
}

contract C is A {
    function foo() public virtual override returns (string memory) {
        return "C";
    }
}

// Linearization: D -> C -> B -> A (direita para esquerda)
contract D is B, C {
    function foo() public override(B, C) returns (string memory) {
        return super.foo(); // Chama C.foo() (próximo na hierarquia)
    }
}
```

### Interfaces

```solidity
// Interface = contrato com apenas assinaturas de funções
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);

    // Events permitidos
    event Transfer(address indexed from, address indexed to, uint256 amount);
}

// Implementar interface
contract MyToken is IERC20 {
    mapping(address => uint256) private _balances;
    uint256 private _totalSupply;

    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }

    function transfer(address to, uint256 amount) external override returns (bool) {
        _balances[msg.sender] -= amount;
        _balances[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
}

// Usar interface para chamar outro contrato
contract TokenUser {
    function transferTokens(address tokenAddress, address to, uint256 amount) public {
        IERC20 token = IERC20(tokenAddress); // Cast para interface
        token.transfer(to, amount);           // Chama função do outro contrato
    }
}
```

---

## 3.8 Peculiaridades Críticas

### 1. Divisão de Inteiros (Sem Float!)

```solidity
// ❌ PEGADINHA: divisão trunca
uint256 a = 5;
uint256 b = 2;
uint256 result = a / b; // = 2, não 2.5!

// ✅ SOLUÇÃO: multiplicar antes de dividir
uint256 percentage = (amount * 100) / total; // %
uint256 precise = (amount * 1e18) / total;   // Usar 18 decimals como "float"
```

### 2. Não Há Null/Undefined

```solidity
// Todos os tipos têm valor default
uint256 x;    // = 0
bool y;       // = false
address z;    // = 0x0000000000000000000000000000000000000000
string s;     // = ""

// ⚠️ Não dá para distinguir "não setado" de "setado para default"!
mapping(address => uint256) balances;

function checkUser(address user) public view returns (uint256) {
    return balances[user]; // Retorna 0 se nunca foi setado OU se balance é 0!
}

// ✅ SOLUÇÃO: usar struct com flag
struct Balance {
    uint256 amount;
    bool exists;
}

mapping(address => Balance) public balances2;
```

### 3. Overflow/Underflow (Antes de 0.8.0)

```solidity
// Solidity < 0.8.0: overflow wraps around silenciosamente!
uint8 x = 255;
x = x + 1; // = 0 (wrap around) - BUG SILENCIOSO!

// Solidity >= 0.8.0: reverte automaticamente
uint8 x = 255;
x = x + 1; // ❌ REVERT: "Arithmetic operation underflowed or overflowed"

// Se você QUER wrap around (raro), use unchecked
unchecked {
    uint8 x = 255;
    x = x + 1; // = 0 (wraps, economiza gas)
}
```

### 4. Arrays length-- é Perigoso

```solidity
// ❌ ANTES: Solidity permitia
uint[] public numbers = [1, 2, 3];
numbers.length--; // Remove último - REMOVED em 0.6.0!

// ✅ AGORA: Use .pop()
numbers.pop();
```

### 5. Delete Não Remove, Zera

```solidity
struct User {
    string name;
    uint256 balance;
}

User public user = User("Alice", 100);

delete user;
// user.name = ""
// user.balance = 0
// Struct ainda existe, apenas zerado!
```

### 6. String Comparison Não é Óbvio

```solidity
// ❌ ERRO: não pode comparar strings diretamente
string memory a = "hello";
string memory b = "hello";
// if (a == b) {} // ❌ NÃO COMPILA!

// ✅ SOLUÇÃO: comparar hashes
if (keccak256(abi.encodePacked(a)) == keccak256(abi.encodePacked(b))) {
    // Strings são iguais
}
```

### 7. Private Não é Secreto!

```solidity
contract Vault {
    uint256 private secretNumber = 42; // ❌ NÃO É SECRETO!
}

// Qualquer um pode ler storage:
// web3.eth.getStorageAt(contractAddress, 0) = "0x2a" (42 em hex)
```

💡 **Pro Tip**: `private` significa "outras contracts não podem acessar via função", mas blockchain é pública!

### 8. Require vs Assert vs Revert

```solidity
// require: validação de inputs (refund gas restante)
function transfer(address to, uint256 amount) public {
    require(to != address(0), "Invalid address");
    require(amount > 0, "Amount must be positive");
    // ...
}

// assert: invariantes que nunca devem falhar (consome todo gas em versões antigas)
function internalCalc(uint256 a, uint256 b) private pure returns (uint256) {
    uint256 c = a + b;
    assert(c >= a); // Overflow check (redundante em 0.8+)
    return c;
}

// revert: erro condicional customizado
function complexLogic() public {
    if (someComplexCondition) {
        revert("Complex condition failed");
    }
}

// Custom errors (0.8.4+, mais eficiente!)
error InsufficientBalance(uint256 available, uint256 required);

function withdraw(uint256 amount) public {
    if (balances[msg.sender] < amount) {
        revert InsufficientBalance({
            available: balances[msg.sender],
            required: amount
        });
    }
}
```

---

## 📖 Glossário Consolidado

**pragma**
> Diretiva de compilador que especifica versão de Solidity.
>
> ```solidity
> pragma solidity ^0.8.0;  // 0.8.0 até <0.9.0
> pragma solidity >=0.8.0 <0.9.0; // Equivalente
> pragma solidity 0.8.19; // Versão exata
> ```

**abi.encode vs abi.encodePacked**
> - `abi.encode`: Padding completo (32 bytes por argumento)
> - `abi.encodePacked`: Tight packing (sem padding, menos gas, mas collision risk)
>
> ```solidity
> abi.encode("A", "B")        // 64 bytes
> abi.encodePacked("A", "B")  // 2 bytes
> ```

**msg.data**
> Calldata completo (function selector + argumentos encoded).
> Útil para proxies e meta-transactions.

**tx.gasprice**
> Preço de gas desta transação (em Wei).
> Definido por quem enviou a transação.

---

## 🔒 Security Checklist: Solidity Basics

- [ ] **Versão de Solidity**: Usar >= 0.8.0 (overflow checks automáticos)
- [ ] **Integer division**: Cuidado com truncamento, multiplicar antes de dividir
- [ ] **Null checks**: Lembrar que tudo tem valor default
- [ ] **String comparison**: Usar keccak256 para comparar
- [ ] **Private variables**: NÃO são secretas na blockchain
- [ ] **Delete**: Zera, não remove - considere se é o comportamento desejado
- [ ] **Array bounds**: Verificar length antes de acessar índice
- [ ] **Mapping defaults**: Retornam 0/false/"", não null

---

## 📝 Exercícios Práticos

### Exercício 1: Otimização de Storage

Otimize este contrato para usar menos storage slots:

```solidity
contract Unoptimized {
    uint256 public count;      // Slot 0
    uint8 public status;       // Slot 1
    uint16 public id;          // Slot 2
    bool public active;        // Slot 3
    address public owner;      // Slot 4
}
```

<details>
<summary>✅ Solução</summary>

```solidity
contract Optimized {
    address public owner;      // Slot 0 (20 bytes)
    uint16 public id;          // Slot 0 (2 bytes) - packed!
    uint8 public status;       // Slot 0 (1 byte) - packed!
    bool public active;        // Slot 0 (1 byte) - packed!
    uint256 public count;      // Slot 1 (não cabe no slot 0)
}

// Original: 5 slots
// Otimizado: 2 slots
// Economia: 3 SSTORE = ~45,000 gas!
```

**Regra**: Agrupe variáveis pequenas para caber em 32 bytes.
</details>

### Exercício 2: Iterable Mapping

Implemente uma estrutura que permita iterar sobre um mapping:

```solidity
contract Exercise {
    // TODO: Implementar mapping iterável de address => uint256
    // Funções necessárias:
    // - set(address key, uint256 value)
    // - get(address key) returns (uint256)
    // - getAll() returns (address[], uint256[])
    // - remove(address key)
}
```

<details>
<summary>✅ Solução</summary>

```solidity
contract IterableMapping {
    mapping(address => uint256) private values;
    mapping(address => uint256) private indexOf; // Index no array + 1 (0 = não existe)
    address[] private keys;

    function set(address key, uint256 value) public {
        if (indexOf[key] == 0) {
            // Novo key
            keys.push(key);
            indexOf[key] = keys.length; // length = index + 1
        }
        values[key] = value;
    }

    function get(address key) public view returns (uint256) {
        return values[key];
    }

    function getAll() public view returns (address[] memory, uint256[] memory) {
        uint256[] memory vals = new uint256[](keys.length);
        for (uint256 i = 0; i < keys.length; i++) {
            vals[i] = values[keys[i]];
        }
        return (keys, vals);
    }

    function remove(address key) public {
        uint256 index = indexOf[key];
        require(index > 0, "Key not found");

        // Move último elemento para posição removida
        address lastKey = keys[keys.length - 1];
        keys[index - 1] = lastKey;
        indexOf[lastKey] = index;

        // Remove último
        keys.pop();
        delete indexOf[key];
        delete values[key];
    }
}
```
</details>

---

## 📚 Recursos Adicionais

### Documentação Essencial
1. **[Solidity Docs](https://docs.soliditylang.org/)** - Documentação oficial completa
2. **[Solidity by Example](https://solidity-by-example.org/)** - Exemplos práticos e concisos
3. **[Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html)** - Convenções de código

### Ferramentas
- **[Remix IDE](https://remix.ethereum.org/)** - IDE online para começar rápido
- **[Solidity Visual Developer](https://marketplace.visualstudio.com/items?itemName=tintinweb.solidity-visual-auditor)** - Plugin VSCode

---

## 🎯 Próximos Passos

Agora que domina Solidity, está pronto para:

→ **Capítulo 4**: Ambiente de Desenvolvimento Profissional
- Hardhat vs Foundry
- Testing frameworks
- Deployment

→ **Capítulo 5**: Design Patterns em Solidity
- Checks-Effects-Interactions
- Proxy patterns
- Factory pattern

---

**Autor**: Baseado no material do ITA Blockchain Club + experiência prática
**Última Atualização**: 2025-11-14
