# Capítulo 6: Testing - Unit, Integration, Fork Tests

> **Para Desenvolvedores Experientes**: Se você vem de TDD, conhece a importância de testes. Em blockchain, teste não é opcional - é questão de sobrevivência financeira. Bugs custam milhões e são irreversíveis. Coverage de 95%+ não é perfeccionismo, é mínimo aceitável.

---

## Índice
- [6.1 Por Que Testing é Crítico](#61-por-que-testing-é-crítico)
- [6.2 Unit Tests com Foundry](#62-unit-tests-com-foundry)
- [6.3 Integration Tests](#63-integration-tests)
- [6.4 Fork Testing](#64-fork-testing)
- [6.5 Fuzzing](#65-fuzzing)
- [6.6 Coverage](#66-coverage)
- [6.7 Test-Driven Development](#67-test-driven-development)

---

## 6.1 Por Que Testing é Crítico

### Diferença de Web2

| Web2 | Blockchain |
|------|------------|
| Bug? Deploy hotfix em minutos | Bug? Milhões perdidos, irreversível |
| Rollback? Fácil | Rollback? Impossível (ou hard fork) |
| Testes? Bom ter | Testes? **OBRIGATÓRIO** |

### Casos Reais

- **The DAO**: $60M roubados - testes teriam detectado reentrancy
- **Parity**: $150M congelados - testes teriam detectado init bug
- **Poly Network**: $600M roubados - testes teriam detectado access control

**Conclusão**: Invista 50% do tempo em testes!

---

## 6.2 Unit Tests com Foundry

### Estrutura Básica

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/Token.sol";

contract TokenTest is Test {
    Token public token;
    address public alice = address(0x1);
    address public bob = address(0x2);

    function setUp() public {
        token = new Token("Test", "TST");
        token.mint(alice, 1000);
    }

    function testTransfer() public {
        vm.prank(alice);
        token.transfer(bob, 100);

        assertEq(token.balanceOf(alice), 900);
        assertEq(token.balanceOf(bob), 100);
    }

    function testCannotTransferMoreThanBalance() public {
        vm.prank(alice);
        vm.expectRevert("Insufficient balance");
        token.transfer(bob, 2000);
    }

    function testEmitsTransferEvent() public {
        vm.prank(alice);

        vm.expectEmit(true, true, false, true);
        emit Transfer(alice, bob, 100);

        token.transfer(bob, 100);
    }
}
```

### Cheatcodes Essenciais

```solidity
function testCheatcodes() public {
    // Alterar msg.sender
    vm.prank(alice);        // Próxima chamada
    vm.startPrank(alice);   // Todas até stopPrank
    vm.stopPrank();

    // Alterar block
    vm.warp(1000);          // block.timestamp
    vm.roll(100);           // block.number

    // ETH
    vm.deal(alice, 10 ether);

    // Expect revert
    vm.expectRevert();
    vm.expectRevert("Error message");
    vm.expectRevert(abi.encodeWithSignature("CustomError()"));

    // Expect emit
    vm.expectEmit(true, true, false, true);
    emit SomeEvent(param1, param2);

    // Mock
    vm.mockCall(target, calldata, returndata);

    // Labels (para debug)
    vm.label(alice, "Alice");
}
```

---

## 6.3 Integration Tests

### Testar Múltiplos Contratos Juntos

```solidity
contract IntegrationTest is Test {
    Token public token;
    Exchange public exchange;
    Staking public staking;

    address public user = address(0x1);

    function setUp() public {
        token = new Token();
        exchange = new Exchange(address(token));
        staking = new Staking(address(token));

        token.mint(user, 1000 ether);
    }

    function testCompleteFlow() public {
        vm.startPrank(user);

        // 1. Approve exchange
        token.approve(address(exchange), 100 ether);

        // 2. Swap no exchange
        exchange.swap(100 ether);

        // 3. Stake tokens recebidos
        uint256 received = token.balanceOf(user);
        token.approve(address(staking), received);
        staking.stake(received);

        // 4. Avançar tempo
        vm.warp(block.timestamp + 30 days);

        // 5. Claim rewards
        staking.claimRewards();

        vm.stopPrank();

        // Verificar estado final
        assertGt(token.balanceOf(user), received, "Should have rewards");
    }
}
```

---

## 6.4 Fork Testing

### Testar Contra Mainnet Real

```solidity
contract ForkTest is Test {
    uint256 mainnetFork;

    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant UNISWAP_ROUTER = 0x...; // Uniswap V3 Router

    function setUp() public {
        // Fork mainnet
        mainnetFork = vm.createFork(vm.envString("MAINNET_RPC_URL"));
        vm.selectFork(mainnetFork);
    }

    function testSwapOnUniswap() public {
        // Dar ETH para test address
        vm.deal(address(this), 10 ether);

        // Swap ETH → USDC no Uniswap REAL
        ISwapRouter router = ISwapRouter(UNISWAP_ROUTER);

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: WETH,
                tokenOut: USDC,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: 1 ether,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });

        router.exactInputSingle{value: 1 ether}(params);

        // Verificar que recebeu USDC
        assertGt(IERC20(USDC).balanceOf(address(this)), 0);
    }
}
```

**Vantagens Fork Testing**:
- ✅ Testa contra contratos reais (Uniswap, Aave, etc.)
- ✅ State real da mainnet
- ✅ Detecta bugs de integração

---

## 6.5 Fuzzing

### Testes com Inputs Aleatórios

```solidity
contract FuzzTest is Test {
    Token public token;

    function setUp() public {
        token = new Token();
    }

    // Foundry roda isso milhares de vezes com valores diferentes
    function testFuzzTransfer(address to, uint256 amount) public {
        // Assume válido
        vm.assume(to != address(0));
        vm.assume(amount <= token.totalSupply());

        address from = address(this);
        token.mint(from, amount);

        uint256 balanceBefore = token.balanceOf(to);
        token.transfer(to, amount);

        // Invariante: balance deve aumentar exatamente amount
        assertEq(token.balanceOf(to), balanceBefore + amount);
    }

    // Fuzzing com múltiplos parâmetros
    function testFuzz_NoOverflow(uint128 a, uint128 b) public pure {
        // Garante que soma não overflow
        uint256 result = uint256(a) + uint256(b);
        assertLe(result, type(uint256).max);
    }
}
```

**Configurar fuzzing** (foundry.toml):
```toml
[fuzz]
runs = 10000  # Quantas vezes rodar cada test
max_test_rejects = 65536  # Max rejects de vm.assume
```

**Rodar**:
```bash
forge test --fuzz-runs 50000
```

---

## 6.6 Coverage

### Medir Coverage

```bash
# Gerar relatório de coverage
forge coverage

# Output exemplo:
# src/Token.sol: 95.2% (40/42 lines)
# src/Exchange.sol: 87.5% (28/32 lines)
```

### Interpretar Coverage

- **<90%**: ❌ Insuficiente, muitos caminhos não testados
- **90-95%**: ⚠️ Aceitável, mas melhorar
- **95%+**: ✅ Bom, maioria dos caminhos cobertos
- **100%**: 🎯 Ideal (mas não garante ausência de bugs!)

### Ignorar Código de Coverage

```solidity
// solhint-disable-next-line
function unreachableCode() private {
    // Código que não deve ser testado
}
```

---

## 6.7 Test-Driven Development (TDD)

### Red-Green-Refactor

```solidity
// 1. RED: Escrever teste que falha
function testStaking() public {
    staking.stake(100);
    assertEq(staking.stakedAmount(address(this)), 100);
}
// ❌ Falha: função stake() não existe

// 2. GREEN: Implementar mínimo para passar
function stake(uint256 amount) public {
    stakedAmounts[msg.sender] = amount;
}
// ✅ Passa

// 3. REFACTOR: Melhorar código
function stake(uint256 amount) public {
    require(amount > 0, "Amount must be positive");
    require(token.balanceOf(msg.sender) >= amount, "Insufficient balance");

    token.transferFrom(msg.sender, address(this), amount);
    stakedAmounts[msg.sender] += amount;

    emit Staked(msg.sender, amount);
}
// ✅ Ainda passa, mas mais robusto
```

---

## 📖 Glossário

**Fuzzing**
> Técnica de teste com inputs aleatórios/semi-aleatórios para encontrar edge cases.

**Fork Testing**
> Testar contra snapshot de mainnet/testnet real, interagindo com contratos existentes.

**Coverage**
> Percentagem de código executado durante testes. Linhas, branches, etc.

**Invariant**
> Propriedade que deve ser SEMPRE verdadeira (ex: totalSupply = sum of balances).

---

## 🔒 Security Checklist

- [ ] Unit tests para CADA função pública
- [ ] Integration tests para fluxos completos
- [ ] Fork tests para integração com protocolos existentes
- [ ] Fuzzing com 10k+ runs
- [ ] Coverage >= 95%
- [ ] Testes para casos de erro (reverts)
- [ ] Testes para eventos
- [ ] Testes para edge cases (0, max uint, etc.)

---

## 📝 Exercício

Escrever testes para este contrato:

```solidity
contract Vault {
    mapping(address => uint256) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw(uint256 amount) public {
        require(balances[msg.sender] >= amount);
        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
}
```

Incluir:
- Unit tests (deposit, withdraw)
- Fuzz tests
- Edge cases (reentrancy? overflow?)
- Coverage 100%

---

## 📚 Recursos

1. **[Foundry Testing Guide](https://book.getfoundry.sh/forge/tests)**
2. **[Echidna](https://github.com/crytic/echidna)** - Fuzzer avançado

---

**Próximo**: Capítulo 7 - Gas Optimization (testar é bom, otimizar é melhor).
