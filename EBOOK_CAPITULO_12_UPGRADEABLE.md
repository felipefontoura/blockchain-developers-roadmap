# Capítulo 12: Upgradeable Contracts e Governança

> **Para Desenvolvedores Experientes**: Em Web2, você faz deploy de bugfix em minutos. Rolling updates, blue-green deployment, canary releases - tudo padrão. Em blockchain, código é imutável. "Code is law" soa ótimo até você encontrar um bug crítico em produção com $100M em TVL. Proxies são o "hack" que permite upgrades sem mudar endereço do contrato.

---

## Índice
- [12.1 O Problema da Imutabilidade](#121-o-problema-da-imutabilidade)
- [12.2 Proxy Pattern Fundamentals](#122-proxy-pattern-fundamentals)
- [12.3 Transparent Proxy](#123-transparent-proxy)
- [12.4 UUPS Proxy](#124-uups-proxy)
- [12.5 Diamond Pattern](#125-diamond-pattern)
- [12.6 Storage Collision](#126-storage-collision)
- [12.7 Governança On-Chain](#127-governança-on-chain)
- [12.8 Trade-offs](#128-trade-offs)

---

## 12.1 O Problema da Imutabilidade

### Imutabilidade: Feature ou Bug?

**Web2 Deployment**:
```bash
# Deploy v1
git push production

# Bug encontrado!
# Fix em 10 minutos
git commit -m "Fix critical bug"
git push production  # ✅ Deploy imediato
```

**Web3 Deployment**:
```bash
# Deploy v1
forge create Contract

# Bug encontrado!
# ❌ Contrato é imutável - não pode mudar código
# ❌ Não pode adicionar features
# ❌ $10M em TVL está preso em código bugado
```

### Casos Reais de Imutabilidade

**Parity Multisig Hack (2017)**:
```solidity
// Bug: função de inicialização era pública
function initWallet(address[] owners) public {
    // ... setup wallet
}

// Attacker chamou initWallet() e se tornou owner
// $150M congelados PERMANENTEMENTE
// Sem upgrade = sem fix
```

**Compound cETH Bug (2021)**:
```
Bug: Distribuição incorreta de COMP tokens
Resultado: $80M distribuídos erroneamente
Solução: Não tinha - teve que convencer users a devolver
```

---

## 12.2 Proxy Pattern Fundamentals

### Como Proxies Funcionam

**Conceito**: Separar **lógica** (implementation) de **dados** (proxy).

```
User → Proxy (armazena dados) → delegatecall → Implementation (lógica)
                                                      ↓
                                              Executa usando storage do Proxy
```

### Delegatecall Explained

**Comparação**:
```solidity
// CALL: Executa no contexto do contrato chamado
contractB.call(data);
// → msg.sender = contractA
// → storage de contractB é modificado

// DELEGATECALL: Executa código do chamado, mas contexto do caller
contractB.delegatecall(data);
// → msg.sender = original caller
// → storage de contractA é modificado (não contractB!)
```

### Proxy Básico

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleProxy {
    address public implementation;
    address public admin;

    constructor(address _implementation) {
        implementation = _implementation;
        admin = msg.sender;
    }

    // Admin pode trocar implementation
    function upgrade(address newImplementation) external {
        require(msg.sender == admin, "Not admin");
        implementation = newImplementation;
    }

    // Fallback: delega todas chamadas para implementation
    fallback() external payable {
        address impl = implementation;

        assembly {
            // Copiar calldata
            calldatacopy(0, 0, calldatasize())

            // Delegatecall para implementation
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)

            // Copiar returndata
            returndatacopy(0, 0, returndatasize())

            // Retornar ou reverter
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    receive() external payable {}
}
```

### Implementation Example

```solidity
contract ImplementationV1 {
    // ⚠️ Storage layout DEVE ser idêntico ao proxy!
    address public implementation; // Slot 0
    address public admin;          // Slot 1

    // Lógica do contrato
    uint256 public value;

    function setValue(uint256 _value) public {
        value = _value;
    }

    function getValue() public view returns (uint256) {
        return value;
    }
}

contract ImplementationV2 {
    // ⚠️ MESMO storage layout!
    address public implementation;
    address public admin;
    uint256 public value;

    // Nova função!
    function setValue(uint256 _value) public {
        value = _value * 2; // Lógica diferente
    }

    function getValue() public view returns (uint256) {
        return value;
    }

    // Feature nova
    function reset() public {
        value = 0;
    }
}
```

**Upgrade Flow**:
```solidity
// Deploy
ImplementationV1 implV1 = new ImplementationV1();
SimpleProxy proxy = new SimpleProxy(address(implV1));

// Users interagem com proxy
IImplementation(address(proxy)).setValue(42);

// Upgrade
ImplementationV2 implV2 = new ImplementationV2();
proxy.upgrade(address(implV2));

// Agora users usam V2, mas dados permanecem!
IImplementation(address(proxy)).getValue(); // Ainda 42
IImplementation(address(proxy)).reset();     // Nova função!
```

---

## 12.3 Transparent Proxy

### Problema: Function Selector Clash

```solidity
// Proxy tem: upgrade(address)
// Implementation tem: upgrade(address) (por coincidência)

// User chama proxy.upgrade() - qual executa?
// → Se proxy, ok
// → Se delega para implementation, admin perde controle!
```

**Solução**: **Transparent Proxy** - se caller é admin, nunca delega.

### Implementação OpenZeppelin

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import "@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol";

// Implementation
contract Box {
    uint256 private value;

    event ValueChanged(uint256 newValue);

    function store(uint256 newValue) public {
        value = newValue;
        emit ValueChanged(newValue);
    }

    function retrieve() public view returns (uint256) {
        return value;
    }
}

// Deploy script
contract Deploy {
    function run() external {
        // 1. Deploy implementation
        Box implementation = new Box();

        // 2. Deploy proxy admin
        ProxyAdmin proxyAdmin = new ProxyAdmin();

        // 3. Deploy proxy
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(
            address(implementation),
            address(proxyAdmin),
            "" // initialization data
        );

        // 4. Users interagem com proxy
        Box(address(proxy)).store(42);
    }
}
```

### Transparent Proxy Logic

```solidity
// Pseudo-código do Transparent Proxy
fallback() external payable {
    if (msg.sender == admin) {
        // Admin: executa funções de admin (upgrade, etc.)
        // NÃO delega
        if (msg.sig == upgradeToAndCall.selector) {
            upgradeToAndCall(...);
        }
    } else {
        // Non-admin: SEMPRE delega
        _delegate(implementation);
    }
}
```

💡 **Pro Tip**: Admin NUNCA pode chamar funções do implementation via proxy (para evitar clash).

---

## 12.4 UUPS Proxy

### UUPS = Universal Upgradeable Proxy Standard

**Diferença do Transparent**:
- **Transparent**: Upgrade logic no **proxy**
- **UUPS**: Upgrade logic no **implementation**

**Vantagem**: Gas mais barato (proxy é mais simples).

**Desvantagem**: Implementation deve herdar upgrade logic.

### Implementação

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract BoxV1 is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    uint256 private value;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize() public initializer {
        __Ownable_init();
        __UUPSUpgradeable_init();
    }

    function store(uint256 newValue) public {
        value = newValue;
    }

    function retrieve() public view returns (uint256) {
        return value;
    }

    // Upgrade logic (somente owner)
    function _authorizeUpgrade(address newImplementation)
        internal
        override
        onlyOwner
    {}
}

contract BoxV2 is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    uint256 private value;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize() public initializer {
        __Ownable_init();
        __UUPSUpgradeable_init();
    }

    function store(uint256 newValue) public {
        value = newValue;
    }

    function retrieve() public view returns (uint256) {
        return value;
    }

    // Nova função em V2
    function increment() public {
        value += 1;
    }

    function _authorizeUpgrade(address newImplementation)
        internal
        override
        onlyOwner
    {}
}
```

### Deploy UUPS

```solidity
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract DeployUUPS {
    function run() external {
        // 1. Deploy implementation
        BoxV1 implementation = new BoxV1();

        // 2. Encode initialization call
        bytes memory data = abi.encodeWithSelector(
            BoxV1.initialize.selector
        );

        // 3. Deploy proxy
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(implementation),
            data
        );

        // 4. Interact via proxy
        BoxV1(address(proxy)).store(42);

        // 5. Upgrade
        BoxV2 implementationV2 = new BoxV2();
        BoxV1(address(proxy)).upgradeToAndCall(
            address(implementationV2),
            "" // no initialization needed
        );

        // 6. Use new features
        BoxV2(address(proxy)).increment();
    }
}
```

### Transparent vs UUPS

| Aspecto | Transparent | UUPS |
|---------|-------------|------|
| **Upgrade logic** | No proxy | No implementation |
| **Gas (user calls)** | Mais caro (if/else) | Mais barato |
| **Gas (deployment)** | Proxy maior | Proxy menor |
| **Risco** | Menor | Se esquecer upgrade logic em V2 = brick! |
| **Flexibilidade** | Menor | Maior (pode customizar upgrade) |
| **OpenZeppelin Support** | ✅ | ✅ |

💡 **Recomendação**: Use UUPS para economizar gas, mas teste MUITO bem!

---

## 12.5 Diamond Pattern (EIP-2535)

### O Problema de Contratos Grandes

```solidity
// Contrato monolítico
contract HugeContract {
    // 100+ funções
    // 50+ state variables
    // Problema: Passa do limite de 24KB de bytecode!
}
```

**Limites do Ethereum**:
- Max contract size: **24KB** de bytecode
- Workaround tradicional: Múltiplos contratos + inheritance
- Diamond: Múltiplos "facets" compartilhando storage

### Diamond Architecture

```
           Diamond (Proxy)
                 │
    ┌────────────┼────────────┐
    │            │            │
 Facet A      Facet B      Facet C
(funções     (funções     (funções
  1-20)       21-40)       41-60)

Todos usam MESMO storage (do Diamond)
```

### Implementação Simplificada

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Diamond {
    // Mapping: function selector → facet address
    mapping(bytes4 => address) public facets;
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    // Adicionar função
    function addFunction(bytes4 selector, address facet) external {
        require(msg.sender == owner);
        facets[selector] = facet;
    }

    // Remover função
    function removeFunction(bytes4 selector) external {
        require(msg.sender == owner);
        delete facets[selector];
    }

    // Delegatecall para facet correto
    fallback() external payable {
        address facet = facets[msg.sig];
        require(facet != address(0), "Function not found");

        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}

// Facet A: User management
contract UserFacet {
    // Storage (DEVE corresponder ao Diamond!)
    mapping(bytes4 => address) public facets; // Slot 0
    address public owner;                     // Slot 1

    // User data
    mapping(address => string) public usernames;

    function setUsername(string calldata name) external {
        usernames[msg.sender] = name;
    }

    function getUsername(address user) external view returns (string memory) {
        return usernames[user];
    }
}

// Facet B: Token management
contract TokenFacet {
    mapping(bytes4 => address) public facets;
    address public owner;
    mapping(address => string) public usernames;

    // Token data
    mapping(address => uint256) public balances;

    function mint(address to, uint256 amount) external {
        balances[to] += amount;
    }
}
```

### Diamond Setup

```solidity
contract DeployDiamond {
    function run() external {
        // 1. Deploy Diamond
        Diamond diamond = new Diamond();

        // 2. Deploy Facets
        UserFacet userFacet = new UserFacet();
        TokenFacet tokenFacet = new TokenFacet();

        // 3. Register functions
        diamond.addFunction(
            UserFacet.setUsername.selector,
            address(userFacet)
        );
        diamond.addFunction(
            UserFacet.getUsername.selector,
            address(userFacet)
        );
        diamond.addFunction(
            TokenFacet.mint.selector,
            address(tokenFacet)
        );

        // 4. Interact
        UserFacet(address(diamond)).setUsername("Alice");
        TokenFacet(address(diamond)).mint(msg.sender, 1000);
    }
}
```

**Quando usar**:
- ✅ Contratos muito grandes (>24KB)
- ✅ Múltiplos módulos independentes
- ✅ Upgrades granulares (trocar apenas 1 facet)

**Quando NÃO usar**:
- ❌ Contratos simples (overhead desnecessário)
- ❌ Primeira vez com proxies (use UUPS/Transparent primeiro)

---

## 12.6 Storage Collision

### O Maior Perigo de Proxies

**Problema**: Proxy e Implementation compartilham storage slots!

```solidity
// Proxy
contract Proxy {
    address public implementation; // Slot 0
    address public admin;          // Slot 1
}

// Implementation V1
contract ImplementationV1 {
    uint256 public value; // ❌ Também usa Slot 0!
    // COLLISION! value sobrescreve implementation address!
}
```

**Resultado**:
```solidity
proxy.setValue(12345);
// Escreve 12345 no slot 0
// Slot 0 = implementation address
// Implementation agora aponta para 0x0000000000000000000000000000000000003039
// Proxy quebrado permanentemente!
```

### Solução 1: Namespace Storage (ERC-1967)

```solidity
contract ERC1967Proxy {
    // Usar slot calculado via hash (collision quase impossível)
    bytes32 private constant IMPLEMENTATION_SLOT =
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
        // = bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1)

    bytes32 private constant ADMIN_SLOT =
        0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;
        // = bytes32(uint256(keccak256('eip1967.proxy.admin')) - 1)

    function _getImplementation() internal view returns (address impl) {
        assembly {
            impl := sload(IMPLEMENTATION_SLOT)
        }
    }

    function _setImplementation(address newImpl) internal {
        assembly {
            sstore(IMPLEMENTATION_SLOT, newImpl)
        }
    }
}
```

**Por que funciona**: Slot `0x360894...` está "muito longe" de slots normais (0, 1, 2...), collision impossível.

### Solução 2: Storage Gap

```solidity
contract ImplementationV1 {
    // Reservar slots do proxy
    address private __gap_implementation;
    address private __gap_admin;

    // Agora sim, dados do implementation
    uint256 public value;
}

contract ImplementationV2 {
    // MESMO layout!
    address private __gap_implementation;
    address private __gap_admin;
    uint256 public value;

    // Novas variáveis sempre no final
    uint256 public newVariable;
}
```

### Solução 3: OpenZeppelin Storage Layout

```solidity
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract MyContract is Initializable {
    // OpenZeppelin cuida do storage layout!
    // Inicializador em vez de constructor
    function initialize(uint256 _value) public initializer {
        value = _value;
    }

    uint256 public value;
}
```

⚠️ **Regras de Ouro**:
1. NUNCA mude ordem de variáveis existentes
2. NUNCA mude tipo de variáveis existentes
3. SEMPRE adicione novas variáveis no FINAL
4. Use OpenZeppelin Upgradeable Contracts

---

## 12.7 Governança On-Chain

### Por Que Governança?

**Problema**: Quem pode fazer upgrade? Um admin centralizado?

**Solução**: **DAO** (Decentralized Autonomous Organization) - token holders votam.

### Governança Simples

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract GovernanceToken is ERC20 {
    constructor() ERC20("Governance Token", "GOV") {
        _mint(msg.sender, 1_000_000 * 10**18);
    }
}

contract SimpleGovernor {
    GovernanceToken public token;

    struct Proposal {
        string description;
        address target;      // Contrato a chamar
        bytes data;          // Calldata
        uint256 forVotes;
        uint256 againstVotes;
        uint256 deadline;
        bool executed;
        mapping(address => bool) voted;
    }

    Proposal[] public proposals;

    uint256 public constant VOTING_PERIOD = 3 days;
    uint256 public constant QUORUM = 100_000 * 10**18; // 10%

    event ProposalCreated(uint256 indexed proposalId, string description);
    event Voted(uint256 indexed proposalId, address indexed voter, bool support, uint256 weight);
    event ProposalExecuted(uint256 indexed proposalId);

    constructor(address _token) {
        token = GovernanceToken(_token);
    }

    // Criar proposta
    function propose(
        string calldata description,
        address target,
        bytes calldata data
    ) external returns (uint256) {
        require(token.balanceOf(msg.sender) >= 1000 * 10**18, "Not enough tokens");

        uint256 proposalId = proposals.length;
        Proposal storage proposal = proposals.push();

        proposal.description = description;
        proposal.target = target;
        proposal.data = data;
        proposal.deadline = block.timestamp + VOTING_PERIOD;

        emit ProposalCreated(proposalId, description);
        return proposalId;
    }

    // Votar
    function vote(uint256 proposalId, bool support) external {
        Proposal storage proposal = proposals[proposalId];

        require(block.timestamp < proposal.deadline, "Voting ended");
        require(!proposal.voted[msg.sender], "Already voted");

        uint256 weight = token.balanceOf(msg.sender);
        require(weight > 0, "No voting power");

        proposal.voted[msg.sender] = true;

        if (support) {
            proposal.forVotes += weight;
        } else {
            proposal.againstVotes += weight;
        }

        emit Voted(proposalId, msg.sender, support, weight);
    }

    // Executar proposta (se passou)
    function execute(uint256 proposalId) external {
        Proposal storage proposal = proposals[proposalId];

        require(block.timestamp >= proposal.deadline, "Voting not ended");
        require(!proposal.executed, "Already executed");
        require(proposal.forVotes > proposal.againstVotes, "Proposal failed");
        require(proposal.forVotes >= QUORUM, "Quorum not reached");

        proposal.executed = true;

        // Execute call
        (bool success, ) = proposal.target.call(proposal.data);
        require(success, "Execution failed");

        emit ProposalExecuted(proposalId);
    }

    // View
    function getProposal(uint256 proposalId) external view returns (
        string memory description,
        uint256 forVotes,
        uint256 againstVotes,
        uint256 deadline,
        bool executed
    ) {
        Proposal storage proposal = proposals[proposalId];
        return (
            proposal.description,
            proposal.forVotes,
            proposal.againstVotes,
            proposal.deadline,
            proposal.executed
        );
    }
}
```

### Upgrade via Governança

```solidity
contract GovernedProxy is UUPSUpgradeable, OwnableUpgradeable {
    SimpleGovernor public governor;

    function initialize(address _governor) public initializer {
        __Ownable_init();
        __UUPSUpgradeable_init();
        governor = SimpleGovernor(_governor);
    }

    // Somente governor pode fazer upgrade
    function _authorizeUpgrade(address newImplementation)
        internal
        override
        onlyOwner
    {
        // Transferir ownership para governor após deploy
    }

    // Proposta para fazer upgrade
    function proposeUpgrade(address newImplementation) external {
        bytes memory data = abi.encodeWithSelector(
            this.upgradeToAndCall.selector,
            newImplementation,
            ""
        );

        governor.propose(
            "Upgrade to new implementation",
            address(this),
            data
        );
    }
}
```

### Timelock

**Problema**: Governança aprova upgrade → executa imediatamente → users não têm tempo de reagir.

**Solução**: **Timelock** - delay obrigatório entre aprovação e execução.

```solidity
contract Timelock {
    uint256 public constant DELAY = 2 days;

    mapping(bytes32 => uint256) public queuedTransactions;

    event QueueTransaction(bytes32 indexed txHash, address target, bytes data, uint256 eta);
    event ExecuteTransaction(bytes32 indexed txHash);

    function queueTransaction(address target, bytes memory data) external returns (bytes32) {
        bytes32 txHash = keccak256(abi.encode(target, data));
        uint256 eta = block.timestamp + DELAY;

        queuedTransactions[txHash] = eta;

        emit QueueTransaction(txHash, target, data, eta);
        return txHash;
    }

    function executeTransaction(address target, bytes memory data) external {
        bytes32 txHash = keccak256(abi.encode(target, data));
        uint256 eta = queuedTransactions[txHash];

        require(eta != 0, "Transaction not queued");
        require(block.timestamp >= eta, "Timelock not expired");

        delete queuedTransactions[txHash];

        (bool success, ) = target.call(data);
        require(success, "Transaction failed");

        emit ExecuteTransaction(txHash);
    }
}
```

💡 **Pro Tip**: Compound, Uniswap, Aave todos usam timelock de 2-7 dias.

---

## 12.8 Trade-offs

### Upgradeable vs Immutable

| Aspecto | Upgradeable | Immutable |
|---------|-------------|-----------|
| **Security** | ❌ Admin pode rug pull | ✅ Trustless |
| **Bug fixes** | ✅ Possível | ❌ Impossível |
| **Gas** | ❌ Mais caro (delegatecall) | ✅ Direto |
| **Complexity** | ❌ Alta | ✅ Simples |
| **Trust** | ❌ Precisa confiar em admin/DAO | ✅ Code is law |
| **Compliance** | ✅ Pode atualizar para leis novas | ❌ Rígido |

### Quando Usar Upgradeable

✅ **Use quando**:
- Protocolo novo (alta chance de bugs)
- Features evoluem rapidamente
- Compliance é necessário
- Governança descentralizada (DAO)

❌ **Não use quando**:
- Contrato é simples e bem testado
- Trustlessness é prioridade máxima
- DeFi com muito TVL (target de hacks)

### Estratégia Híbrida

```solidity
// Core (imutável)
contract Core {
    // Lógica crítica, bem auditada
    // NUNCA muda
}

// Periphery (upgradeable)
contract Periphery is UUPSUpgradeable {
    Core public immutable core;

    // Features experimentais
    // Pode ser upgrade sem afetar Core
}
```

**Exemplo**: Uniswap V3 - core imutável, periphery upgradeable.

---

## 📖 Glossário

**Proxy Pattern**
> Separar storage (proxy) de lógica (implementation) para permitir upgrades.
> **Analogia**: Proxy = database, Implementation = application server.

**Delegatecall**
> Executar código de contrato B usando storage de contrato A.
> **Por que existe**: Permite proxies funcionarem.

**Transparent Proxy**
> Proxy que nunca delega chamadas do admin (evita function clash).
> **Trade-off**: Mais gas para users.

**UUPS**
> Upgrade logic no implementation (não no proxy).
> **Trade-off**: Mais barato, mas risco de brick se esquecer upgrade logic.

**Diamond Pattern**
> Múltiplos "facets" compartilhando mesmo storage.
> **Quando usar**: Contratos > 24KB.

**Storage Collision**
> Proxy e implementation sobrescrevem mesmo storage slot.
> **Proteção**: ERC-1967 namespaced storage.

**Timelock**
> Delay obrigatório entre aprovar e executar transação.
> **Por que**: Dar tempo para users reagirem.

**Quorum**
> Mínimo de votos necessários para proposta passar.
> **Exemplo**: 10% do supply total.

**DAO**
> Decentralized Autonomous Organization - governança por token holders.

---

## 🔒 Security Checklist: Upgradeable Contracts

### Storage Layout
- [ ] NUNCA mudar ordem de variáveis
- [ ] NUNCA mudar tipo de variáveis
- [ ] SEMPRE adicionar novas variáveis no final
- [ ] Usar OpenZeppelin storage gaps
- [ ] Testar storage compatibility antes de upgrade

### Initialization
- [ ] Usar `initializer` em vez de `constructor`
- [ ] Desabilitar initializers em implementation (`_disableInitializers()`)
- [ ] Proteger initializer contra re-inicialização
- [ ] Nunca deixar initialize() sem access control

### Upgrade Process
- [ ] Timelock de pelo menos 24-48h
- [ ] Multi-sig ou DAO para aprovar upgrades
- [ ] Test upgrade em testnet primeiro
- [ ] Verificar implementation antes de deploy
- [ ] Comunicar upgrade para comunidade

### Testing
- [ ] Unit tests para implementation
- [ ] Integration tests com proxy
- [ ] Fork tests simulando upgrade
- [ ] Verificar storage layout compatibility
- [ ] Test de governança (propostas, votos, timelock)

### Documentation
- [ ] Documentar storage layout
- [ ] Explicar processo de upgrade
- [ ] Listar admin/governor addresses
- [ ] Changelog de cada versão

---

## 📝 Exercícios Práticos

### Exercício 1: UUPS Upgradeable Token

**Objetivo**: Criar token ERC-20 upgradeable.

**V1**: ERC-20 básico
**V2**: Adicionar pausable, burnable

**Requisitos**:
- Deploy com UUPS proxy
- Upgrade de V1 para V2
- Verificar que balances permanecem após upgrade
- Testar novas funções (pause, burn)

<details>
<summary>💡 Dica</summary>

Use OpenZeppelin:
```solidity
import "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
```

Lembre de `initializer` modifier e `__ERC20_init("Name", "SYM")`.
</details>

---

### Exercício 2: DAO com Governança

**Objetivo**: Criar DAO que governa upgradeable contract.

**Componentes**:
1. Governance Token (ERC-20)
2. Governor (propostas, votos)
3. Timelock (2 dias)
4. Proxy upgradeable

**Flow**:
1. Criar proposta de upgrade
2. Votação (3 dias)
3. Se aprovado → Queue no timelock
4. Após 2 dias → Execute upgrade

<details>
<summary>💡 Estrutura</summary>

```solidity
GovernanceToken → Governor → Timelock → Proxy
                     ↓
                 Proposals & Votes
```

Use OpenZeppelin Governor contracts como base.
</details>

---

### Exercício 3: Storage Collision Attack

**Objetivo**: Entender storage collision na prática (educacional).

**Setup**:
1. Deploy proxy "vulnerável" (sem namespaced storage)
2. Deploy implementation que causa collision
3. Demonstrar como proxy "quebra"

**Aprendizado**: Ver POR QUE ERC-1967 storage é necessário.

⚠️ **Nota**: Apenas educational, nunca deploy em mainnet!

---

## 📚 Recursos Adicionais

### Documentação

1. **[OpenZeppelin Upgradeable Contracts](https://docs.openzeppelin.com/contracts/4.x/upgradeable)**
   - Transparent, UUPS, storage layout
   - Best practices

2. **[EIP-1967: Proxy Storage Slots](https://eips.ethereum.org/EIPS/eip-1967)**
   - Standard de storage namespacing

3. **[EIP-2535: Diamond Standard](https://eips.ethereum.org/EIPS/eip-2535)**
   - Multi-facet proxy pattern

### Ferramentas

- **[Hardhat Upgrades Plugin](https://docs.openzeppelin.com/upgrades-plugins/1.x/)**
- **[Foundry Upgrades](https://github.com/foundry-rs/foundry)**
- **[Gnosis Safe](https://gnosis-safe.io/)** - Multi-sig para admin

### Deep Dives

- **[Trail of Bits: Upgradeability](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/)**
- **[Consensys: Proxy Patterns](https://consensys.github.io/smart-contract-best-practices/development-recommendations/solidity-specific/proxy-libraries/)**

---

## 🎯 Próximos Passos

**Você agora entende**:
- ✅ Por que proxies existem (imutabilidade vs upgradeability)
- ✅ Transparent vs UUPS vs Diamond
- ✅ Storage collision e como evitar
- ✅ Governança on-chain (DAOs)
- ✅ Timelock para segurança
- ✅ Trade-offs de upgradeability

**Próximo Capítulo**: [Capítulo 13 - Front-end Integration](./EBOOK_CAPITULO_13_FRONTEND.md)
- Ethers.js vs Web3.js
- Conectar wallet (MetaMask)
- React hooks para Web3
- Ler e escrever dados on-chain
- Error handling

**Aplicação Prática**:
Combine conhecimentos:
- Cap 10 (DeFi) + Cap 11 (Oracles) + Cap 12 (Upgradeable) = DeFi protocol production-ready
- DAO que governa DEX upgradeable
- Lending protocol com oracle Chainlink e governança

💡 **Projeto Sugerido**: DEX upgradeable governado por DAO + Timelock + UI (Cap 13).

---

**Autor**: Baseado em roadmap ITA Blockchain Club + OpenZeppelin patterns
**Última Atualização**: 2025-11-14
