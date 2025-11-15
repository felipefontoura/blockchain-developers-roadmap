# Capítulo 5: Design Patterns em Solidity

> **Para Desenvolvedores Experientes**: Se você conhece Singleton, Factory, Observer em programação tradicional, vai reconhecer padrões familiares aqui. Mas blockchain adiciona restrições únicas: imutabilidade, gas costs, e adversários maliciosos. Estes patterns não são apenas elegantes - são questão de segurança e economia.

---

## Índice
- [5.1 Checks-Effects-Interactions](#51-checks-effects-interactions)
- [5.2 Pull over Push (Withdrawal Pattern)](#52-pull-over-push-withdrawal-pattern)
- [5.3 Access Control Patterns](#53-access-control-patterns)
- [5.4 Factory Pattern](#54-factory-pattern)
- [5.5 Proxy Patterns](#55-proxy-patterns)
- [5.6 Circuit Breaker (Pausable)](#56-circuit-breaker-pausable)
- [5.7 Rate Limiting](#57-rate-limiting)
- [5.8 Commit-Reveal](#58-commit-reveal)

---

## 5.1 Checks-Effects-Interactions

### O Pattern Mais Importante em Solidity

**Regra de Ouro**: SEMPRE siga esta ordem:
1. **Checks**: Validações (require, assert)
2. **Effects**: Modificar state
3. **Interactions**: Chamar contratos externos

### Por Que Existe

Previne **reentrancy attacks** (visto no Cap 8).

### Implementação

```solidity
// ✅ CORRETO: Checks-Effects-Interactions
function withdraw(uint256 amount) public {
    // 1. CHECKS
    require(balances[msg.sender] >= amount, "Insufficient balance");
    require(amount > 0, "Amount must be positive");

    // 2. EFFECTS (modifica state ANTES de external call)
    balances[msg.sender] -= amount;

    // 3. INTERACTIONS (external call por último)
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");
}

// ❌ ERRADO: Interactions antes de Effects
function badWithdraw(uint256 amount) public {
    require(balances[msg.sender] >= amount);

    // ❌ External call ANTES de atualizar state
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success);

    balances[msg.sender] -= amount; // Tarde demais!
}
```

**Analogia Web2**: Como commit de transação SQL - todas mudanças locais primeiro, depois comunicação externa.

---

## 5.2 Pull over Push (Withdrawal Pattern)

### Problema: Push Payments

```solidity
// ❌ ANTI-PATTERN: Push (enviar automaticamente)
contract BadAuction {
    address[] public bidders;

    function refundAll() public {
        for (uint i = 0; i < bidders.length; i++) {
            payable(bidders[i]).transfer(refunds[bidders[i]]);
        }
    }

    // Problemas:
    // 1. Gas limit (muitos bidders)
    // 2. DoS se um bidder revert
    // 3. Custo alto para quem chama
}
```

### Solução: Pull Pattern

```solidity
// ✅ PATTERN: Pull (cada um retira)
contract GoodAuction {
    mapping(address => uint256) public pendingWithdrawals;

    function withdraw() public {
        uint256 amount = pendingWithdrawals[msg.sender];
        require(amount > 0, "No funds");

        pendingWithdrawals[msg.sender] = 0;

        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
    }
}
```

**Vantagens**:
- ✅ Sem gas limit issues
- ✅ DoS resistant
- ✅ Usuário paga próprio gas

---

## 5.3 Access Control Patterns

### Ownable Pattern

```solidity
contract Ownable {
    address private _owner;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    constructor() {
        _owner = msg.sender;
        emit OwnershipTransferred(address(0), _owner);
    }

    modifier onlyOwner() {
        require(msg.sender == _owner, "Not owner");
        _;
    }

    function transferOwnership(address newOwner) public onlyOwner {
        require(newOwner != address(0), "Invalid address");
        emit OwnershipTransferred(_owner, newOwner);
        _owner = newOwner;
    }

    function renounceOwnership() public onlyOwner {
        emit OwnershipTransferred(_owner, address(0));
        _owner = address(0);
    }
}
```

### Role-Based Access Control (RBAC)

```solidity
import "@openzeppelin/contracts/access/AccessControl.sol";

contract MultiRole is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
    bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");

    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    function mint(address to, uint256 amount) public onlyRole(MINTER_ROLE) {
        // Apenas minters podem mintar
    }

    function pause() public onlyRole(PAUSER_ROLE) {
        // Apenas pausers podem pausar
    }
}
```

---

## 5.4 Factory Pattern

### Criar Contratos de Forma Padronizada

```solidity
contract TokenFactory {
    event TokenCreated(address indexed token, address indexed owner);

    address[] public allTokens;
    mapping(address => address[]) public userTokens;

    function createToken(
        string memory name,
        string memory symbol
    ) public returns (address) {
        // Deploy novo contrato
        MyToken token = new MyToken(name, symbol, msg.sender);

        address tokenAddress = address(token);
        allTokens.push(tokenAddress);
        userTokens[msg.sender].push(tokenAddress);

        emit TokenCreated(tokenAddress, msg.sender);

        return tokenAddress;
    }

    function getUserTokens(address user) public view returns (address[] memory) {
        return userTokens[user];
    }
}

contract MyToken {
    string public name;
    string public symbol;
    address public owner;

    constructor(string memory _name, string memory _symbol, address _owner) {
        name = _name;
        symbol = _symbol;
        owner = _owner;
    }
}
```

**Uso Real**: Uniswap usa factory para criar liquidity pools.

---

## 5.5 Proxy Patterns

### Por Que Proxies?

**Problema**: Smart contracts são imutáveis.
**Solução**: Proxy pattern permite "upgrade" sem mudar endereço.

### Transparent Proxy Pattern

```solidity
// Proxy (nunca muda)
contract TransparentProxy {
    address public implementation;
    address public admin;

    constructor(address _implementation) {
        implementation = _implementation;
        admin = msg.sender;
    }

    modifier onlyAdmin() {
        require(msg.sender == admin, "Not admin");
        _;
    }

    function upgrade(address newImplementation) external onlyAdmin {
        implementation = newImplementation;
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

// Implementation V1
contract ImplementationV1 {
    uint256 public value;

    function setValue(uint256 _value) public {
        value = _value;
    }
}

// Implementation V2 (upgrade)
contract ImplementationV2 {
    uint256 public value;

    function setValue(uint256 _value) public {
        value = _value * 2; // Nova lógica!
    }
}
```

**Como funciona**:
1. Users chamam Proxy
2. Proxy usa `delegatecall` para Implementation
3. Code de Implementation executa, mas usa storage de Proxy
4. Admin pode trocar Implementation

⚠️ **Cuidado**: Storage layout deve ser compatível!

💡 **Pro Tip**: Use OpenZeppelin Upgradeable Contracts (muito complexo fazer manualmente).

---

## 5.6 Circuit Breaker (Pausable)

### Parar Contrato em Emergência

```solidity
contract Pausable {
    bool private _paused;
    address private _owner;

    event Paused(address account);
    event Unpaused(address account);

    constructor() {
        _owner = msg.sender;
        _paused = false;
    }

    modifier whenNotPaused() {
        require(!_paused, "Contract is paused");
        _;
    }

    modifier whenPaused() {
        require(_paused, "Contract is not paused");
        _;
    }

    modifier onlyOwner() {
        require(msg.sender == _owner);
        _;
    }

    function pause() public onlyOwner whenNotPaused {
        _paused = true;
        emit Paused(msg.sender);
    }

    function unpause() public onlyOwner whenPaused {
        _paused = false;
        emit Unpaused(msg.sender);
    }
}

// Uso
contract MyToken is Pausable {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) public whenNotPaused {
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}
```

**Quando usar**: Contratos que guardam muito valor (pode pausar se exploit detectado).

---

## 5.7 Rate Limiting

### Limitar Ações por Tempo

```solidity
contract RateLimited {
    uint256 public constant RATE_LIMIT = 10; // Max 10 por período
    uint256 public constant PERIOD = 1 days;

    mapping(address => uint256) public actionCount;
    mapping(address => uint256) public periodStart;

    modifier rateLimit() {
        if (block.timestamp >= periodStart[msg.sender] + PERIOD) {
            // Novo período
            periodStart[msg.sender] = block.timestamp;
            actionCount[msg.sender] = 0;
        }

        require(actionCount[msg.sender] < RATE_LIMIT, "Rate limit exceeded");
        actionCount[msg.sender]++;
        _;
    }

    function sensitiveAction() public rateLimit {
        // Apenas 10 vezes por dia
    }
}
```

---

## 5.8 Commit-Reveal

### Esconder Intenção Contra Front-Running

```solidity
contract CommitReveal {
    struct Commit {
        bytes32 commit;
        uint256 block;
        bool revealed;
    }

    mapping(address => Commit) public commits;
    uint256 public constant REVEAL_DELAY = 10; // 10 blocos

    // Fase 1: Commit (hash secreto)
    function commit(bytes32 _commit) public {
        commits[msg.sender] = Commit({
            commit: _commit,
            block: block.number,
            revealed: false
        });
    }

    // Fase 2: Reveal (após delay)
    function reveal(uint256 value, bytes32 salt) public {
        Commit storage c = commits[msg.sender];

        require(c.commit != bytes32(0), "No commit");
        require(!c.revealed, "Already revealed");
        require(block.number >= c.block + REVEAL_DELAY, "Too early");

        bytes32 hash = keccak256(abi.encodePacked(value, salt));
        require(hash == c.commit, "Invalid reveal");

        c.revealed = true;

        // Usar value (já não pode ser front-run)
        processValue(value);
    }

    function processValue(uint256 value) private {
        // Lógica que usa o value revelado
    }
}
```

**Uso**: Leilões, votações, jogos on-chain.

---

## 📖 Glossário

**Delegatecall**
> External call que executa código de outro contrato usando storage do caller.
> Usado em proxies para separar logic de data.

**Storage Layout**
> Ordem das variáveis de state em storage slots.
> Crítico em proxies - deve ser compatível entre versões.

**Assembly (Yul)**
> Linguagem de baixo nível para EVM.
> Usada em proxies para controle fino de delegatecall.

---

## 🔒 Security Checklist

- [ ] Seguir Checks-Effects-Interactions religiosamente
- [ ] Preferir Pull over Push para pagamentos
- [ ] Access control em TODAS funções sensíveis
- [ ] Se usar proxy, garantir storage layout compatível
- [ ] Circuit breaker para contratos de alto valor
- [ ] Rate limiting para prevenir spam
- [ ] Commit-reveal para prevenir front-running

---

## 📝 Exercício

Implemente um sistema de votação com:
- Commit-reveal (votos secretos)
- Access control (apenas membros votam)
- Pausable (pode pausar votação)
- Pull pattern (distribuir rewards)

---

## 📚 Recursos

1. **[OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)** - Implementations prontas
2. **[Solidity Patterns](https://fravoll.github.io/solidity-patterns/)** - Catálogo completo

---

**Próximo**: Capítulo 6 - Testing para garantir que patterns funcionam corretamente.
