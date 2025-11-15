# Capítulo 8: Security - Top 10 Vulnerabilidades

> **Para Desenvolvedores Experientes**: Se você já lidou com XSS, SQL injection, ou CSRF em Web2, sabe que segurança não é opcional. Em blockchain, os stakes são LITERALMENTE mais altos - bugs custam milhões de dólares reais, são públicos, e geralmente irreversíveis. Este capítulo pode salvar sua carreira (e muito dinheiro).

---

## ⚠️ Aviso Crítico

**Bugs em smart contracts já custaram BILHÕES**:
- The DAO: $60 milhões (2016)
- Parity Wallet: $150 milhões (2017)
- Ronin Bridge: $625 milhões (2022)
- Wormhole: $326 milhões (2022)

**Diferença de Web2**:
- ❌ Não há "hotfix" rápido
- ❌ Não há "rollback" de produção
- ❌ Código é público (atacantes veem tudo)
- ❌ Dinheiro real está em risco

**Este capítulo pode prevenir você de aparecer em [Rekt News](https://rekt.news/).**

---

## Índice
- [8.1 Reentrancy - The DAO Hack](#81-reentrancy---the-dao-hack)
- [8.2 Integer Overflow/Underflow](#82-integer-overflowunderflow)
- [8.3 Access Control Vulnerabilities](#83-access-control-vulnerabilities)
- [8.4 Delegatecall Exploits](#84-delegatecall-exploits)
- [8.5 Front-Running e MEV](#85-front-running-e-mev)
- [8.6 Randomness Manipulation](#86-randomness-manipulation)
- [8.7 Denial of Service (DoS)](#87-denial-of-service-dos)
- [8.8 Unchecked External Calls](#88-unchecked-external-calls)
- [8.9 tx.origin Authentication](#89-txorigin-authentication)
- [8.10 Flash Loan Attacks](#810-flash-loan-attacks)

---

## 8.1 Reentrancy - The DAO Hack

### O Ataque Mais Famoso da História

**The DAO (2016)**: $60 milhões roubados, levou a hard fork do Ethereum (ETH vs ETC).

### Como Funciona

**Reentrancy**: Atacante chama função que faz external call, e "reentra" antes da primeira chamada terminar.

### Código Vulnerável

```solidity
// ❌ VULNERÁVEL a reentrancy
contract VulnerableBank {
    mapping(address => uint256) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() public {
        uint256 balance = balances[msg.sender];

        // PROBLEMA: external call ANTES de atualizar state!
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success);

        // State atualizado DEPOIS da call
        balances[msg.sender] = 0;
    }
}
```

### Exploit

```solidity
contract Attack {
    VulnerableBank public bank;
    uint256 public constant AMOUNT = 1 ether;

    constructor(address _bank) {
        bank = VulnerableBank(_bank);
    }

    // 1. Depositar 1 ETH
    function attack() external payable {
        bank.deposit{value: AMOUNT}();
        bank.withdraw(); // 2. Iniciar primeiro withdraw
    }

    // 3. Fallback é chamado quando recebe ETH
    receive() external payable {
        if (address(bank).balance >= AMOUNT) {
            bank.withdraw(); // 4. REENTRA em withdraw!
        }
    }
}
```

**Fluxo do ataque**:
```
1. Attack.attack() → deposita 1 ETH
2. bank.withdraw()
   ├─ balance = 1 ETH
   ├─ msg.sender.call{value: 1 ETH}  ← envia ETH
   │   └─ Attack.receive() acionado
   │       └─ bank.withdraw() REENTRA!
   │           ├─ balance = 1 ETH (ainda não foi zerado!)
   │           ├─ msg.sender.call{value: 1 ETH}
   │           └─ balances[msg.sender] = 0
   └─ balances[msg.sender] = 0

Resultado: Atacante recebeu 2 ETH, tinha apenas 1 ETH depositado!
Pode repetir até esvaziar contrato.
```

### Solução 1: Checks-Effects-Interactions Pattern

```solidity
// ✅ SEGURO: Checks-Effects-Interactions
contract SecureBank {
    mapping(address => uint256) public balances;

    function withdraw() public {
        uint256 balance = balances[msg.sender];

        // CHECKS
        require(balance > 0, "Insufficient balance");

        // EFFECTS (atualizar state ANTES de external call)
        balances[msg.sender] = 0;

        // INTERACTIONS (external call por último)
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success);
    }
}
```

### Solução 2: ReentrancyGuard (OpenZeppelin)

```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract SecureBankWithGuard is ReentrancyGuard {
    mapping(address => uint256) public balances;

    function withdraw() public nonReentrant { // ← Modifier previne reentrancy
        uint256 balance = balances[msg.sender];
        require(balance > 0);

        balances[msg.sender] = 0;

        (bool success, ) = msg.sender.call{value: balance}("");
        require(success);
    }
}
```

**Como ReentrancyGuard funciona**:
```solidity
// Simplified version
abstract contract ReentrancyGuard {
    uint256 private _status;

    modifier nonReentrant() {
        require(_status != 2, "ReentrancyGuard: reentrant call");
        _status = 2; // Locked

        _;

        _status = 1; // Unlocked
    }
}
```

### 🔒 Checklist: Prevenir Reentrancy

- [ ] Seguir Checks-Effects-Interactions SEMPRE
- [ ] Atualizar state ANTES de external calls
- [ ] Usar `nonReentrant` em funções que fazem external calls
- [ ] Preferir `transfer()/send()` sobre `.call()` quando possível
- [ ] Testar com mock contracts que tentam reentrancy

---

## 8.2 Integer Overflow/Underflow

### Problema (Antes de Solidity 0.8.0)

```solidity
// Solidity < 0.8.0
uint8 public count = 255;

function increment() public {
    count = count + 1; // Wrap around: 255 → 0 (SILENCIOSAMENTE!)
}

function decrement() public {
    count = count - 1; // Underflow: 0 → 255
}
```

### Exploits Reais

**BeautyChain (BEC) Token (2018)**: $900 milhões de market cap destruído por overflow.

```solidity
// ❌ Código vulnerável (simplificado)
function batchTransfer(address[] recipients, uint256 value) public {
    uint256 amount = recipients.length * value; // OVERFLOW!

    require(balances[msg.sender] >= amount); // Bypass se overflow

    for (uint i = 0; i < recipients.length; i++) {
        balances[recipients[i]] += value;
    }
}

// Exploit: recipients.length = 2, value = 2^255
// amount = 2 * 2^255 = 2^256 = 0 (overflow!)
// require(balances[msg.sender] >= 0) ✅ passa!
// Atacante cria tokens infinitos
```

### Solução 1: Solidity >= 0.8.0

```solidity
// Solidity 0.8.0+ reverte automaticamente
uint8 public count = 255;

function increment() public {
    count = count + 1; // ❌ REVERT: "Arithmetic operation overflowed"
}
```

### Solução 2: SafeMath (Legacy, pré-0.8.0)

```solidity
import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract LegacyContract {
    using SafeMath for uint256;

    uint256 public count;

    function increment() public {
        count = count.add(1); // Reverte se overflow
    }

    function decrement() public {
        count = count.sub(1); // Reverte se underflow
    }
}
```

### Quando Desabilitar Checks (Raro)

```solidity
// Se você QUER wrap around E tem certeza que é seguro
function optimizedLoop() public {
    unchecked {
        for (uint256 i = 0; i < 1000; i++) {
            // i++ não vai overflow em 1000 iterações
            // Economiza ~20 gas por iteração
        }
    }
}
```

⚠️ **Warning**: Só use `unchecked` se você TEM CERTEZA que não vai overflow!

---

## 8.3 Access Control Vulnerabilities

### Problema: Funções Públicas Não-Protegidas

```solidity
// ❌ VULNERÁVEL: qualquer um pode chamar!
contract VulnerableToken {
    mapping(address => uint256) public balances;

    function mint(address to, uint256 amount) public {
        balances[to] += amount; // Qualquer um pode mintar!
    }
}
```

### Solução: Modifiers de Acesso

```solidity
// ✅ SEGURO: apenas owner pode mintar
contract SecureToken {
    address public owner;
    mapping(address => uint256) public balances;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    function mint(address to, uint256 amount) public onlyOwner {
        balances[to] += amount;
    }
}
```

### Role-Based Access Control (RBAC)

```solidity
import "@openzeppelin/contracts/access/AccessControl.sol";

contract AdvancedToken is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");

    mapping(address => uint256) public balances;

    constructor() {
        // Admin role
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);

        // Grant minter role to deployer
        _grantRole(MINTER_ROLE, msg.sender);
    }

    function mint(address to, uint256 amount) public onlyRole(MINTER_ROLE) {
        balances[to] += amount;
    }

    function burn(address from, uint256 amount) public onlyRole(BURNER_ROLE) {
        balances[from] -= amount;
    }

    // Admin pode dar/remover roles
    function grantMinter(address account) public onlyRole(DEFAULT_ADMIN_ROLE) {
        _grantRole(MINTER_ROLE, account);
    }
}
```

### Exploit Real: Parity Wallet (2017)

**Bug**: Função `initWallet()` podia ser chamada por qualquer um DEPOIS do deploy.

```solidity
// ❌ Simplificação do bug
contract ParityWallet {
    address[] public owners;

    function initWallet(address[] _owners) public {
        owners = _owners; // Qualquer um podia "re-inicializar"!
    }
}

// Atacante chamou:
// wallet.initWallet([attackerAddress])
// Agora atacante é owner → roubou fundos
```

**Solução**: Inicializar no constructor OU usar flag `initialized`:
```solidity
bool private initialized;

function initWallet(address[] _owners) public {
    require(!initialized, "Already initialized");
    initialized = true;
    owners = _owners;
}
```

---

## 8.4 Delegatecall Exploits

### Como Delegatecall Funciona

```solidity
// contract A chama contract B via delegatecall
// Código de B executa, mas usa STORAGE de A!

contract A {
    uint256 public x; // Slot 0

    function callB(address b) public {
        b.delegatecall(abi.encodeWithSignature("setX(uint256)", 100));
        // B.setX() executa, mas modifica A.x!
    }
}

contract B {
    uint256 public x; // Também slot 0

    function setX(uint256 _x) public {
        x = _x; // Modifica storage do CALLER (A)!
    }
}
```

### Exploit: Storage Collision

```solidity
// ❌ VULNERÁVEL
contract Proxy {
    address public implementation; // Slot 0
    address public owner;          // Slot 1

    function upgrade(address newImpl) public {
        require(msg.sender == owner);
        implementation = newImpl;
    }

    fallback() external payable {
        address impl = implementation;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}

contract MaliciousImpl {
    address public attacker; // Slot 0 (alinha com implementation!)

    function attack() public {
        attacker = msg.sender; // Sobrescreve Proxy.implementation!
    }
}

// Ataque:
// 1. Deploy MaliciousImpl
// 2. Proxy.upgrade(maliciousImpl)
// 3. maliciousImpl.attack() via proxy
// 4. Agora Proxy.implementation = atacante!
// 5. Atacante controla proxy
```

### Solução: Storage Gaps e Padrões Corretos

```solidity
// ✅ SEGURO: Transparent Proxy Pattern (OpenZeppelin)
contract TransparentProxy {
    bytes32 private constant IMPLEMENTATION_SLOT =
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    bytes32 private constant ADMIN_SLOT =
        0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;

    // Usa storage slots aleatórios (calculados com keccak256)
    // Previne collision
}
```

💡 **Pro Tip**: Use OpenZeppelin Proxy contracts, não implemente do zero!

---

## 8.5 Front-Running e MEV

### O que é Front-Running

**Front-running**: Atacante vê transação pendente na mempool, envia transação com gas maior para executar ANTES.

**Exemplo**:
```
1. Alice envia: buyToken(100 ETH) com gasPrice = 50 gwei
   → Na mempool (pública)

2. Bob vê transação de Alice
3. Bob envia: buyToken(50 ETH) com gasPrice = 100 gwei
   → Executada ANTES de Alice

4. Preço sobe
5. Alice compra mais caro
6. Bob vende com lucro
```

### Tipos de MEV (Maximal Extractable Value)

**1. Front-running**: Executar antes
**2. Back-running**: Executar depois
**3. Sandwich Attack**: Executar antes E depois

**Sandwich Attack Example**:
```
1. Vítima: swap 100 ETH → USDC (grande trade)
2. Atacante vê na mempool
3. Atacante: compra USDC (preço sobe)
4. Vítima: compra USDC (preço alto, slippage)
5. Atacante: vende USDC (lucro)
```

### Proteções

#### 1. Commit-Reveal Scheme

```solidity
contract CommitReveal {
    mapping(address => bytes32) public commits;

    // Fase 1: Commit (esconde intenção)
    function commit(bytes32 hash) public {
        commits[msg.sender] = hash;
    }

    // Fase 2: Reveal (depois de commit period)
    function reveal(uint256 value, bytes32 salt) public {
        bytes32 hash = keccak256(abi.encodePacked(value, salt));
        require(commits[msg.sender] == hash, "Invalid reveal");

        // Executar ação com value
        // ...
    }
}
```

#### 2. Slippage Protection (DEXs)

```solidity
function swap(
    uint256 amountIn,
    uint256 amountOutMin // ← Slippage protection
) public {
    uint256 amountOut = calculateSwap(amountIn);
    require(amountOut >= amountOutMin, "Slippage too high");
    // ...
}
```

#### 3. Flashbots (Ethereum)

- Transações privadas (não vão para mempool público)
- Enviadas diretamente para miners/validators
- Previne front-running

---

## 8.6 Randomness Manipulation

### Problema: Fontes de Randomness Inseguras

```solidity
// ❌ VULNERÁVEL: previsível!
function badRandom() public view returns (uint256) {
    return uint256(keccak256(abi.encodePacked(
        block.timestamp,  // Minerador controla (~15s precision)
        block.difficulty, // Previsível
        msg.sender        // Conhecido
    )));
}

// Minerador pode:
// 1. Calcular resultado antes de minerar
// 2. Descartar bloco se resultado não favorece ele
// 3. Repetir até conseguir resultado favorável
```

### Exploit em Loteria

```solidity
// ❌ VULNERÁVEL
contract VulnerableLottery {
    address[] public players;

    function enter() public payable {
        require(msg.value == 0.1 ether);
        players.push(msg.sender);
    }

    function pickWinner() public {
        uint256 index = uint256(keccak256(abi.encodePacked(
            block.timestamp
        ))) % players.length;

        address winner = players[index];
        payable(winner).transfer(address(this).balance);
    }
}

// Atacante pode:
// - Calcular se vai ganhar ANTES de chamar pickWinner()
// - Só chamar se garantir vitória
```

### Solução: Chainlink VRF

```solidity
import "@chainlink/contracts/src/v0.8/VRFConsumerBase.sol";

contract SecureLottery is VRFConsumerBase {
    bytes32 internal keyHash;
    uint256 internal fee;
    uint256 public randomResult;

    constructor()
        VRFConsumerBase(
            0x..., // VRF Coordinator
            0x...  // LINK Token
        )
    {
        keyHash = 0x...;
        fee = 0.1 * 10**18; // 0.1 LINK
    }

    function getRandomNumber() public returns (bytes32 requestId) {
        require(LINK.balanceOf(address(this)) >= fee);
        return requestRandomness(keyHash, fee);
    }

    // Callback do Chainlink
    function fulfillRandomness(bytes32 requestId, uint256 randomness)
        internal
        override
    {
        randomResult = randomness;
        // Usar randomness para escolher vencedor
    }
}
```

**Por que é seguro?**:
- Oracle descentralizado
- Não pode ser manipulado
- Provável fair (VRF = Verifiable Random Function)

---

## 8.7 Denial of Service (DoS)

### DoS por Gas Limit

```solidity
// ❌ VULNERÁVEL: loop ilimitado
contract VulnerableAuction {
    address[] public bidders;

    function bid() public payable {
        bidders.push(msg.sender);
    }

    // ❌ PROBLEMA: pode exceder gas limit!
    function refundAll() public {
        for (uint256 i = 0; i < bidders.length; i++) {
            payable(bidders[i]).transfer(1 ether);
        }
    }
}

// Se 10,000 bidders, refundAll() vai custar milhões de gas
// → Excede block gas limit (30M)
// → Função não pode ser executada!
```

### Solução: Pull over Push

```solidity
// ✅ SEGURO: cada um retira seu refund
contract SecureAuction {
    mapping(address => uint256) public refunds;

    function bid() public payable {
        refunds[msg.sender] += msg.value;
    }

    // Cada bidder retira individualmente
    function withdraw() public {
        uint256 amount = refunds[msg.sender];
        refunds[msg.sender] = 0;
        payable(msg.sender).transfer(amount);
    }
}
```

### DoS por Revert Malicioso

```solidity
// ❌ VULNERÁVEL
contract VulnerableDistributor {
    address[] public recipients;

    function distribute() public {
        for (uint256 i = 0; i < recipients.length; i++) {
            payable(recipients[i]).transfer(1 ether);
        }
    }
}

// Atacante cria contrato que reverte ao receber ETH
contract MaliciousRecipient {
    receive() external payable {
        revert(); // Sempre reverte!
    }
}

// distribute() vai SEMPRE reverter quando chegar no atacante
// → Ninguém recebe fundos!
```

### Solução: Ignorar Falhas ou Pull Pattern

```solidity
// ✅ Opção 1: Ignorar falhas
function distribute() public {
    for (uint256 i = 0; i < recipients.length; i++) {
        (bool success, ) = recipients[i].call{value: 1 ether}("");
        // Ignora se falhar, continua para próximo
    }
}

// ✅ Opção 2: Pull pattern (melhor)
mapping(address => uint256) public balances;

function claim() public {
    uint256 amount = balances[msg.sender];
    balances[msg.sender] = 0;
    payable(msg.sender).transfer(amount);
}
```

---

## 8.8 Unchecked External Calls

### Problema: Ignorar Retorno de .call()

```solidity
// ❌ VULNERÁVEL: não verifica retorno
function unsafeTransfer(address payable to, uint256 amount) public {
    to.call{value: amount}(""); // Se falhar, continua silenciosamente!
}

// Dinheiro pode ser perdido se call falhar
```

### Solução: Sempre Verificar Retorno

```solidity
// ✅ SEGURO
function safeTransfer(address payable to, uint256 amount) public {
    (bool success, ) = to.call{value: amount}("");
    require(success, "Transfer failed");
}

// ✅ Alternativa: usar transfer() (reverte automaticamente)
function transfer(address payable to, uint256 amount) public {
    to.transfer(amount); // Reverte se falhar
}
```

**Diferenças**:
- `.call()`: retorna bool, você decide se reverter
- `.transfer()`: reverte automaticamente, mas gas limit = 2300
- `.send()`: retorna bool, gas limit = 2300 (não use, deprecated)

---

## 8.9 tx.origin Authentication

### Problema: Usar tx.origin para Autenticação

```solidity
// ❌ VULNERÁVEL
contract VulnerableWallet {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function transfer(address to, uint256 amount) public {
        require(tx.origin == owner, "Not owner"); // ❌ ERRO!
        payable(to).transfer(amount);
    }
}
```

### Exploit: Phishing Attack

```solidity
contract PhishingAttack {
    VulnerableWallet public wallet;

    constructor(address _wallet) {
        wallet = VulnerableWallet(_wallet);
    }

    function attack() public {
        // Owner chama esta função (por engano/phishing)
        wallet.transfer(address(this), 1 ether);
        // tx.origin = owner ✅ passa!
        // Atacante rouba 1 ETH
    }
}
```

**Fluxo**:
```
Owner → PhishingAttack.attack()
        └─ VulnerableWallet.transfer()
            └─ tx.origin = Owner ✅ (passa!)
            └─ msg.sender = PhishingAttack ❌ (mas não checado!)
```

### Solução: Sempre Use msg.sender

```solidity
// ✅ SEGURO
function transfer(address to, uint256 amount) public {
    require(msg.sender == owner, "Not owner"); // ✅ CORRETO!
    payable(to).transfer(amount);
}
```

---

## 8.10 Flash Loan Attacks

### O que são Flash Loans

**Flash Loan**: Empréstimo que deve ser pago de volta NA MESMA TRANSAÇÃO.

```solidity
// Pseudocódigo
function flashLoan(uint256 amount) public {
    uint256 balanceBefore = token.balanceOf(address(this));

    // 1. Emprestar
    token.transfer(msg.sender, amount);

    // 2. Callback (borrower faz o que quiser)
    IFlashLoanReceiver(msg.sender).executeOperation(amount);

    // 3. Verificar que foi pago de volta
    uint256 balanceAfter = token.balanceOf(address(this));
    require(balanceAfter >= balanceBefore, "Loan not repaid");
}
```

### Como são Usados para Ataques

**Típico Attack Flow**:
```
1. Flash loan de 10,000 ETH
2. Manipular preço em DEX A
3. Arbitrar contra DEX B
4. Repagar flash loan + fee
5. Lucrar diferença
```

### Exemplo Real: bZx Attack (2020)

```
1. Flash loan: 10,000 ETH da dYdX
2. Usou para comprar wBTC na Uniswap (preço sobe)
3. Shortou wBTC na bZx (usando preço inflado da Uniswap)
4. Vendeu wBTC (preço volta ao normal)
5. Lucro: $350,000
```

### Proteção: Price Oracles Descentralizados

```solidity
// ❌ VULNERÁVEL: preço de uma única fonte
function getPrice() public view returns (uint256) {
    return uniswap.getPrice(); // Pode ser manipulado!
}

// ✅ SEGURO: TWAP (Time-Weighted Average Price)
import "@uniswap/v2-periphery/contracts/libraries/UniswapV2OracleLibrary.sol";

function getPrice() public view returns (uint256) {
    // Média de preço dos últimos N blocos
    // Não pode ser manipulado em uma transação
    return oracle.getTWAP(1 hours);
}

// ✅ Ainda melhor: Chainlink Price Feeds
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

function getPrice() public view returns (uint256) {
    (, int256 price, , ,) = priceFeed.latestRoundData();
    return uint256(price);
}
```

---

## 🔒 Security Checklist Consolidado

### Antes de Fazer Deploy

- [ ] **Auditoria de código**: Self-audit com checklist abaixo
- [ ] **Testes**: Coverage > 95%
- [ ] **Fuzzing**: Rodar milhares de testes aleatórios
- [ ] **Auditoria profissional**: Para contratos de alto valor
- [ ] **Bug bounty**: Oferecer recompensa para encontrar bugs
- [ ] **Testnet**: Testar em testnet por pelo menos 1 semana
- [ ] **Time-lock**: Implementar para mudanças críticas
- [ ] **Multi-sig**: Para funções de admin
- [ ] **Monitoring**: Setup alertas para atividades suspeitas

### Checklist por Vulnerabilidade

#### Reentrancy
- [ ] Seguir Checks-Effects-Interactions
- [ ] Usar `nonReentrant` modifier
- [ ] Atualizar state ANTES de external calls

#### Overflow/Underflow
- [ ] Usar Solidity >= 0.8.0
- [ ] Se usar `unchecked`, documentar POR QUE é seguro

#### Access Control
- [ ] Todas funções sensíveis têm modifier de acesso
- [ ] Usar `Ownable` ou `AccessControl` (OpenZeppelin)
- [ ] Inicialização protegida

#### Delegatecall
- [ ] Evitar se possível
- [ ] Se usar, garantir storage layout compatível
- [ ] Usar OpenZeppelin Proxy patterns

#### Front-Running
- [ ] Slippage protection em swaps
- [ ] Commit-reveal para ações sensíveis
- [ ] Considerar Flashbots para mainnet

#### Randomness
- [ ] NUNCA usar block.timestamp/difficulty
- [ ] Usar Chainlink VRF

#### DoS
- [ ] Evitar loops ilimitados
- [ ] Pull over Push pattern
- [ ] Usar `.call()` em vez de `.transfer()` para arrays

#### External Calls
- [ ] Sempre verificar retorno de `.call()`
- [ ] Ou usar `.transfer()` (reverte automático)

#### tx.origin
- [ ] NUNCA usar para autenticação
- [ ] Sempre usar `msg.sender`

#### Flash Loans
- [ ] Usar TWAP ou Chainlink para preços
- [ ] Não confiar em preço de uma única fonte

---

## 📝 Exercício Prático: Encontre os Bugs

```solidity
// ATENÇÃO: Este contrato tem 5 vulnerabilidades!
// Encontre todas antes de ver a solução.

contract VulnerableVault {
    mapping(address => uint256) public balances;
    address public owner;
    bool public initialized;

    function init(address _owner) public {
        owner = _owner;
    }

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw(uint256 amount) public {
        require(balances[msg.sender] >= amount);

        payable(msg.sender).call{value: amount}("");

        balances[msg.sender] -= amount;
    }

    function transferOwnership(address newOwner) public {
        require(tx.origin == owner);
        owner = newOwner;
    }

    function emergencyWithdraw() public {
        require(msg.sender == owner);

        for (uint256 i = 0; i < userList.length; i++) {
            payable(userList[i]).transfer(balances[userList[i]]);
            balances[userList[i]] = 0;
        }
    }
}
```

<details>
<summary>✅ Vulnerabilidades e Soluções</summary>

**Bug 1: Inicialização Não Protegida**
```solidity
// ❌ Qualquer um pode chamar init() novamente!
function init(address _owner) public {
    require(!initialized, "Already initialized");
    initialized = true;
    owner = _owner;
}
```

**Bug 2: Reentrancy em withdraw()**
```solidity
// ❌ External call ANTES de atualizar balance
function withdraw(uint256 amount) public {
    require(balances[msg.sender] >= amount);

    // Checks-Effects-Interactions!
    balances[msg.sender] -= amount; // ← Mover para ANTES da call

    (bool success, ) = payable(msg.sender).call{value: amount}("");
    require(success);
}
```

**Bug 3: Unchecked Call Return**
```solidity
// ❌ Não verifica se call() teve sucesso
(bool success, ) = payable(msg.sender).call{value: amount}("");
require(success, "Transfer failed"); // ← Adicionar require
```

**Bug 4: tx.origin Authentication**
```solidity
// ❌ Vulnerável a phishing
function transferOwnership(address newOwner) public {
    require(msg.sender == owner); // ← Usar msg.sender, não tx.origin
    owner = newOwner;
}
```

**Bug 5: DoS em emergencyWithdraw()**
```solidity
// ❌ Loop pode exceder gas limit
// ✅ Usar pull pattern:
mapping(address => bool) public canWithdraw;

function enableEmergencyWithdraw() public {
    require(msg.sender == owner);
    canWithdraw[msg.sender] = true;
}

function emergencyWithdraw() public {
    require(canWithdraw[msg.sender]);
    uint256 amount = balances[msg.sender];
    balances[msg.sender] = 0;
    payable(msg.sender).transfer(amount);
}
```

</details>

---

## 📚 Recursos Adicionais

### Security Tools

1. **[Slither](https://github.com/crytic/slither)** - Static analyzer
   ```bash
   pip3 install slither-analyzer
   slither .
   ```

2. **[Mythril](https://github.com/ConsenSys/mythril)** - Symbolic execution
   ```bash
   pip3 install mythril
   myth analyze contract.sol
   ```

3. **[Echidna](https://github.com/crytic/echidna)** - Fuzzer
   ```bash
   echidna-test contract.sol
   ```

### Learning Resources

- **[Smart Contract Weakness Classification](https://swcregistry.io/)** - Registry de vulnerabilidades
- **[Consensys Best Practices](https://consensys.github.io/smart-contract-best-practices/)** - Guia completo
- **[Rekt News](https://rekt.news/)** - Post-mortems de hacks
- **[Secureum](https://secureum.substack.com/)** - Security auditing bootcamp

### Auditors

Para contratos de alto valor, considere auditoria profissional:
- Trail of Bits
- OpenZeppelin
- ConsenSys Diligence
- Certik
- PeckShield

---

## 🎯 Próximos Passos

Security é jornada contínua. Continue aprendendo:

→ **Capítulo 17**: Auditoria Profissional
- Como fazer self-audit
- Como ler relatórios de auditoria
- Bug bounty programs

→ **Capítulo 6**: Testing Avançado
- Fuzzing com Foundry
- Fork testing
- Coverage 100%

---

**Autor**: Baseado em análise de hacks reais e best practices da indústria
**Última Atualização**: 2025-11-14

**⚠️ IMPORTANTE**: Este capítulo cobre vulnerabilidades comuns, mas NÃO substitui auditoria profissional para contratos de alto valor!
