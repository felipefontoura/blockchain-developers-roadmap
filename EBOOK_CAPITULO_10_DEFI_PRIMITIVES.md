# Capítulo 10: DeFi Primitives - DEX, Lending, Staking

> **Para Desenvolvedores Experientes**: Se você entende APIs REST, bancos de dados relacionais e lógica de negócio, DeFi é simplesmente isso - mas em código imutável e público. DEXs são order books sem servidor central. Lending é SQL JOIN entre credores e devedores, sem banco intermediário. A diferença? Tudo é trustless, auditable, e se você errar, milhões podem ser perdidos.

---

## Índice
- [10.1 AMM - Automated Market Makers](#101-amm---automated-market-makers)
- [10.2 Constant Product Formula](#102-constant-product-formula)
- [10.3 Liquidity Pools](#103-liquidity-pools)
- [10.4 Implementando um DEX Simplificado](#104-implementando-um-dex-simplificado)
- [10.5 Lending Protocols](#105-lending-protocols)
- [10.6 Staking Contracts](#106-staking-contracts)
- [10.7 Security em DeFi](#107-security-em-defi)

---

## 10.1 AMM - Automated Market Makers

### Comparação: Order Book vs AMM

**Web2 / CEX (Centralized Exchange)**:
```
Order Book:
- Sellers: 1 ETH por $2000
- Buyers: 1 ETH por $1995
- Match engine: Encontra melhor preço
- Centralizado: Exchange controla tudo
```

**Web3 / DEX (Decentralized Exchange)**:
```
AMM (Automated Market Maker):
- Liquidity Pool: 100 ETH + 200,000 USDC
- Fórmula matemática calcula preço
- Sem order book, sem matchmaking
- Smart contract controla tudo
```

### Por Que AMM Existe?

**Problemas do Order Book On-Chain**:
1. ❌ **Gas caro**: Cada ordem = transação = gas
2. ❌ **Latência**: Blocos de 12s, não milissegundos
3. ❌ **Liquidez fragmentada**: Precisa muitos makers/takers ativos

**Solução AMM**:
1. ✅ **Liquidez sempre disponível**: Pool sempre tem tokens
2. ✅ **Preço determinístico**: Fórmula matemática, não humanos
3. ✅ **Permissionless**: Qualquer um pode adicionar liquidez

---

## 10.2 Constant Product Formula

### A Fórmula Mágica: x * y = k

```
x = Reserva do Token A
y = Reserva do Token B
k = Constante (produto sempre igual)

Exemplo:
100 ETH * 200,000 USDC = 20,000,000 (k)
```

### Como Funciona o Swap

**Cenário**: Comprar 1 ETH com USDC

```
Estado Inicial:
x = 100 ETH
y = 200,000 USDC
k = 20,000,000

Após Swap (comprar 1 ETH):
x' = 99 ETH (você remove 1 ETH)
y' = ? USDC

Fórmula: x' * y' = k
99 * y' = 20,000,000
y' = 202,020.20 USDC

Você paga: 202,020 - 200,000 = 2,020 USDC
Preço: ~2,020 USDC por ETH
```

### Price Impact

**Quanto maior o trade, pior o preço**:

```solidity
// Trade pequeno (0.1 ETH)
100 * 200,000 = k
99.9 * y' = 20,000,000
y' = 200,200
Custo: 200 USDC (2,000/ETH)

// Trade grande (10 ETH)
100 * 200,000 = k
90 * y' = 20,000,000
y' = 222,222
Custo: 22,222 USDC (2,222/ETH) ← Pior preço!
```

💡 **Pro Tip**: Price impact é como "slippage" em order books, mas matematicamente previsível.

---

## 10.3 Liquidity Pools

### Adicionar Liquidez

**Problema**: Como decidir quanto de cada token adicionar?

**Solução**: Proporcional às reservas atuais!

```solidity
// Pool atual: 100 ETH + 200,000 USDC
// Ratio: 1 ETH = 2,000 USDC

// Adicionar liquidez:
// Opção 1: 1 ETH → precisa 2,000 USDC
// Opção 2: 5,000 USDC → precisa 2.5 ETH
```

### LP Tokens

**Analogia Web2**: LP tokens são como "shares" de um fundo de investimento.

```solidity
// Alice adiciona 10 ETH + 20,000 USDC
// Bob adiciona 5 ETH + 10,000 USDC

Total Pool: 15 ETH + 30,000 USDC
Alice: 66.67% do pool (10/15 ETH)
Bob: 33.33% do pool (5/15 ETH)

LP Tokens:
Alice: 10,000 LP tokens
Bob: 5,000 LP tokens
Total: 15,000 LP tokens
```

### Remover Liquidez

```solidity
// Alice quer remover liquidez
// Pool atual: 150 ETH + 300,000 USDC (após fees acumulados)

Alice tem: 10,000 LP tokens (66.67%)
Alice recebe:
- 100 ETH (66.67% de 150)
- 200,000 USDC (66.67% de 300,000)
```

### Impermanent Loss

⚠️ **Warning**: Maior conceito a entender em AMMs!

**Cenário**:
```
T0 (Deposita):
- 1 ETH = $2,000
- Deposita: 10 ETH + 20,000 USDC
- Total valor: $40,000

T1 (ETH sobe para $4,000):
- Pool rebalanceia via arbitrage
- Nova composição: ~7.07 ETH + 28,280 USDC
- Seu share: 7.07 ETH + 28,280 USDC = $56,560

Se tivesse HODL:
- 10 ETH * $4,000 = $40,000
- 20,000 USDC = $20,000
- Total: $60,000

Impermanent Loss: $60,000 - $56,560 = $3,440 (8.6%)
```

**Por que "impermanent"?** Se preço voltar para $2,000, loss desaparece.

**Compensação**: Fees de trading (0.3% por swap).

---

## 10.4 Implementando um DEX Simplificado

### Contrato Base

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract SimpleDEX is ERC20 {
    IERC20 public immutable token0;
    IERC20 public immutable token1;

    uint256 public reserve0;
    uint256 public reserve1;

    event Swap(address indexed user, uint256 amountIn, uint256 amountOut, address tokenIn);
    event AddLiquidity(address indexed provider, uint256 amount0, uint256 amount1, uint256 liquidity);
    event RemoveLiquidity(address indexed provider, uint256 amount0, uint256 amount1, uint256 liquidity);

    constructor(address _token0, address _token1) ERC20("LP Token", "LP") {
        token0 = IERC20(_token0);
        token1 = IERC20(_token1);
    }

    // Adicionar liquidez
    function addLiquidity(uint256 amount0, uint256 amount1) external returns (uint256 liquidity) {
        token0.transferFrom(msg.sender, address(this), amount0);
        token1.transferFrom(msg.sender, address(this), amount1);

        // Primeira adição de liquidez
        if (totalSupply() == 0) {
            liquidity = sqrt(amount0 * amount1);
        } else {
            // Proporcional às reservas
            liquidity = min(
                (amount0 * totalSupply()) / reserve0,
                (amount1 * totalSupply()) / reserve1
            );
        }

        require(liquidity > 0, "Insufficient liquidity minted");

        _mint(msg.sender, liquidity);

        _update();

        emit AddLiquidity(msg.sender, amount0, amount1, liquidity);
    }

    // Remover liquidez
    function removeLiquidity(uint256 liquidity) external returns (uint256 amount0, uint256 amount1) {
        uint256 balance0 = token0.balanceOf(address(this));
        uint256 balance1 = token1.balanceOf(address(this));

        amount0 = (liquidity * balance0) / totalSupply();
        amount1 = (liquidity * balance1) / totalSupply();

        require(amount0 > 0 && amount1 > 0, "Insufficient liquidity burned");

        _burn(msg.sender, liquidity);

        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);

        _update();

        emit RemoveLiquidity(msg.sender, amount0, amount1, liquidity);
    }

    // Swap token0 → token1
    function swap(uint256 amountIn, uint256 minAmountOut) external returns (uint256 amountOut) {
        require(amountIn > 0, "Insufficient input");

        // Transfer input token
        token0.transferFrom(msg.sender, address(this), amountIn);

        // Calculate output (constant product formula)
        // (x + dx) * (y - dy) = x * y
        // dy = y * dx / (x + dx)

        uint256 amountInWithFee = amountIn * 997; // 0.3% fee
        amountOut = (reserve1 * amountInWithFee) / (reserve0 * 1000 + amountInWithFee);

        require(amountOut >= minAmountOut, "Slippage too high");
        require(amountOut > 0, "Insufficient output");

        // Transfer output token
        token1.transfer(msg.sender, amountOut);

        _update();

        emit Swap(msg.sender, amountIn, amountOut, address(token0));
    }

    // Atualizar reservas
    function _update() private {
        reserve0 = token0.balanceOf(address(this));
        reserve1 = token1.balanceOf(address(this));
    }

    // Funções auxiliares
    function sqrt(uint256 x) private pure returns (uint256) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        uint256 y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
        return y;
    }

    function min(uint256 a, uint256 b) private pure returns (uint256) {
        return a < b ? a : b;
    }

    // View functions
    function getReserves() external view returns (uint256, uint256) {
        return (reserve0, reserve1);
    }

    function getAmountOut(uint256 amountIn) external view returns (uint256) {
        require(amountIn > 0, "Invalid amount");
        uint256 amountInWithFee = amountIn * 997;
        return (reserve1 * amountInWithFee) / (reserve0 * 1000 + amountInWithFee);
    }
}
```

### Como Usar

```solidity
// Deploy
SimpleDEX dex = new SimpleDEX(address(tokenA), address(tokenB));

// Adicionar liquidez inicial
tokenA.approve(address(dex), 100 ether);
tokenB.approve(address(dex), 200_000 ether);
dex.addLiquidity(100 ether, 200_000 ether);

// Swap
tokenA.approve(address(dex), 1 ether);
uint256 minOut = dex.getAmountOut(1 ether) * 99 / 100; // 1% slippage tolerance
dex.swap(1 ether, minOut);
```

---

## 🔍 Estudo de Caso: Uniswap V2 - DEX em Produção

### Uniswap V2 Core

**Contratos**: UniswapV2Pair, UniswapV2Factory, UniswapV2Router02
**TVL**: ~$4B+ (todos os pools combinados)
**Volume**: $500B+ all-time
**Endereço Factory**: [0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f](https://etherscan.io/address/0x5c69bee701ef814a2b6a3edd4b1652cb9cc5aa6f)

### Arquitetura: Core vs Periphery

**Diferença fundamental**: Uniswap separa lógica essencial de conveniências

```
Core (UniswapV2Pair):
- Minimal, immutable
- Apenas funções essenciais
- Nenhuma verificação de slippage
- Difícil de usar diretamente

Periphery (UniswapV2Router02):
- Helper functions
- Slippage protection
- Path routing (A → B → C)
- User-friendly
```

### Nossa Implementação vs Uniswap V2

**SimpleDEX (Tutorial)**:
```solidity
contract SimpleDEX is ERC20 {
    IERC20 public immutable token0;
    IERC20 public immutable token1;

    function swap(uint256 amountIn, uint256 minAmountOut) external {
        // Tudo em 1 contrato
        // Fácil de entender
        // Menos flexível
    }
}
```

**Uniswap V2 (Produção)**:
```solidity
// CORE: UniswapV2Pair.sol (apenas o essencial)
contract UniswapV2Pair is ERC20 {
    // Sem amountIn/amountOut explícitos!
    // Usa balance difference
    function swap(
        uint amount0Out,
        uint amount1Out,
        address to,
        bytes calldata data
    ) external {
        // 1. Transfere tokens ANTES de calcular
        if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out);
        if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out);

        // 2. Flash swap callback (se data != empty)
        if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);

        // 3. Verifica que x * y >= k (após)
        uint balance0 = IERC20(_token0).balanceOf(address(this));
        uint balance1 = IERC20(_token1).balanceOf(address(this));

        uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;

        require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');

        // Constant product check (com fee)
        uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
        uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
        require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');

        _update(balance0, balance1);
    }
}

// PERIPHERY: UniswapV2Router02.sol (user-friendly)
contract UniswapV2Router02 {
    // Função que users realmente usam
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,  // Slippage protection
        address[] calldata path,  // Multi-hop: ETH → USDC → DAI
        address to,
        uint deadline  // Time limit
    ) external returns (uint[] memory amounts) {
        // 1. Calcula amounts para cada hop
        amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);

        // 2. Slippage check
        require(amounts[amounts.length - 1] >= amountOutMin, 'INSUFFICIENT_OUTPUT_AMOUNT');

        // 3. Transfer first token to pair
        TransferHelper.safeTransferFrom(path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]);

        // 4. Execute swaps
        _swap(amounts, path, to);
    }
}
```

### Diferenças-Chave: Tutorial → Produção

| Aspecto | SimpleDEX (Tutorial) | Uniswap V2 (Produção) | Por Quê |
|---------|----------------------|-----------------------|---------|
| **Arquitetura** | Monolítico (1 contrato) | Core + Periphery | Core imutável, Router upgradeável |
| **Swap Input** | `amountIn` explícito | Balance difference | Permite flash swaps |
| **Slippage** | Verificado no contrato | Verificado no router | Core não opina sobre preços |
| **Multi-hop** | ❌ Não suporta | ✅ A → B → C | Routing otimizado |
| **Flash Loans** | ❌ Não | ✅ Flash swaps nativos | Receive before pay |
| **Deadline** | ❌ Não | ✅ Sim | Previne stuck transactions |
| **Gas Optimization** | Básico | Extremo | Milhões em volume = cada gas importa |
| **WETH handling** | Manual | Auto unwrap/wrap | UX: Usar ETH nativo |

### Flash Swaps - Killer Feature

**O que são**: Receber tokens ANTES de pagar!

```solidity
// Exemplo: Flash swap no Uniswap V2
contract FlashSwapper {
    function executeFlashSwap() external {
        address pair = IUniswapV2Factory(FACTORY).getPair(WETH, USDC);

        // 1. Pedir 1000 USDC (sem pagar ainda!)
        bytes memory data = abi.encode(WETH, msg.sender);
        IUniswapV2Pair(pair).swap(0, 1000e6, address(this), data);

        // 2. Uniswap chama nosso callback aqui ↓
    }

    // Callback obrigatório
    function uniswapV2Call(address sender, uint amount0, uint amount1, bytes calldata data) external {
        // Temos 1000 USDC aqui!
        // Fazer algo lucrativo...

        // Devolver 1000 USDC + fee (0.3%)
        uint amountToRepay = amount1 * 1003 / 1000;
        IERC20(USDC).transfer(msg.sender, amountToRepay);
    }
}
```

**Casos de uso**:
- Arbitrage sem capital inicial
- Liquidações em lending protocols
- Refinanciamento de dívidas

### Factory Pattern - Criando Pools

**SimpleDEX**: Deploy manual de cada pool

**Uniswap V2**: Factory cria pools automaticamente

```solidity
contract UniswapV2Factory {
    mapping(address => mapping(address => address)) public getPair;
    address[] public allPairs;

    function createPair(address tokenA, address tokenB) external returns (address pair) {
        require(tokenA != tokenB, 'IDENTICAL_ADDRESSES');

        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        require(token0 != address(0), 'ZERO_ADDRESS');
        require(getPair[token0][token1] == address(0), 'PAIR_EXISTS');

        // Deploy novo par usando CREATE2 (endereço determinístico!)
        bytes memory bytecode = type(UniswapV2Pair).creationCode;
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        assembly {
            pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }

        IUniswapV2Pair(pair).initialize(token0, token1);

        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // Reverse mapping
        allPairs.push(pair);

        emit PairCreated(token0, token1, pair, allPairs.length);
    }
}
```

**Por que CREATE2?** Endereço previsível sem deploying!

```solidity
// Qualquer um pode calcular endereço do pair sem chamada on-chain
function pairFor(address factory, address tokenA, address tokenB) internal pure returns (address pair) {
    (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
    pair = address(uint160(uint256(keccak256(abi.encodePacked(
        hex'ff',
        factory,
        keccak256(abi.encodePacked(token0, token1)),
        hex'96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f' // init code hash
    )))));
}
```

### Otimizações de Gas Avançadas

```solidity
// 1. STORAGE vs MEMORY
// ❌ Nossa versão: Reler storage
reserve0 = token0.balanceOf(address(this));
reserve1 = token1.balanceOf(address(this));

// ✅ Uniswap: Cache em memory, escreve 1x no final
uint112 _reserve0 = reserve0;  // SLOAD (2100 gas)
uint112 _reserve1 = reserve1;
// ... cálculos com _reserve0, _reserve1 (memory)
reserve0 = newReserve0;  // SSTORE (5000 gas) - 1x só
reserve1 = newReserve1;

// 2. PACKING de storage
// Nossa versão: 2 slots (2 * 20k gas)
uint256 public reserve0;
uint256 public reserve1;

// Uniswap: 1 slot! (1 * 20k gas)
uint112 private reserve0;  // 112 bits suficiente
uint112 private reserve1;
uint32  private blockTimestampLast;  // Bonus: timestamp cabe no mesmo slot!

// 3. Unchecked math (Solidity < 0.8)
// Overflow checks custam gas
// Uniswap usa SafeMath seletivamente
```

### TWAP Oracles Integrados

```solidity
// Uniswap V2 automaticamente trackeia preços
uint public price0CumulativeLast;
uint public price1CumulativeLast;
uint32 public blockTimestampLast;

function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
    uint32 blockTimestamp = uint32(block.timestamp % 2**32);
    uint32 timeElapsed = blockTimestamp - blockTimestampLast;

    if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
        // * never overflows, + overflow is desired
        price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
        price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
    }

    reserve0 = uint112(balance0);
    reserve1 = uint112(balance1);
    blockTimestampLast = blockTimestamp;
}
```

**Uso**: Lending protocols usam isso como oracle!

### Minimum Liquidity Lock

```solidity
// UniswapV2Pair: Primeira liquidez
uint public constant MINIMUM_LIQUIDITY = 10**3;

if (_totalSupply == 0) {
    liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
    _mint(address(0), MINIMUM_LIQUIDITY); // Lock permanente!
} else {
    liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
}
```

**Por quê?** Prevenir ataque de inflação do primeiro LP:
- Sem lock: Alice adiciona 1 wei ETH + 1 wei USDC → recebe 1 LP token
- Alice transfere 1000 ETH pro pool diretamente
- Bob adiciona liquidez → recebe 0 LP tokens (arredonda pra baixo!)
- Alice rouba liquidez do Bob

**Com MINIMUM_LIQUIDITY**: Primeira mint queima 1000 wei permanentemente, tornando ataque caro.

### Lições para Produção

1. **Separe Core de Periphery**: Core imutável, Router pode evoluir
2. **Use balance difference, não input amounts**: Permite flash swaps
3. **Factory pattern + CREATE2**: Endereços determinísticos, sem deploy centralizado
4. **TWAP integrado**: Outros protocolos usam como oracle
5. **Storage packing**: uint112 + uint112 + uint32 = 1 slot = 15k gas saved
6. **Minimum liquidity**: Queimar primeiros LP tokens previne ataques
7. **Skim/Sync functions**: Lidar com donations e fees on transfer tokens
8. **Checks após actions**: Permite flash swaps, mas valida no final

### Comparação Uniswap V2 vs V3

| Feature | V2 | V3 | Quando Usar |
|---------|----|----|-------------|
| **Liquidez** | Full range | Concentrated | V3 para stablecoins, V2 para volatile |
| **LP positions** | Fungível (ERC-20) | NFT (ERC-721) | V2 mais simples para composability |
| **Gas (swap)** | ~100k | ~130k | V2 mais barato |
| **Capital efficiency** | Baixa | Alta | V3 para LPs profissionais |
| **Complexidade** | Simples | Complexo | V2 para começar |

---

## 10.5 Lending Protocols

### Como Lending Funciona

**Analogia Web2**: Lending protocol é como banco, mas algorítmico e sem humanos decidindo.

```
Banco Tradicional:
User deposita $1000 → Banco paga 2% APY
Banco empresta $900 → Cobra 5% APR
Banco fica com 3% spread

DeFi Lending:
User deposita 1000 USDC → Pool paga 4% APY (algorítmico)
Borrower pega emprestado 900 USDC → Paga 6% APR
Protocol fica com 2% (ou distribui para governance)
```

### Over-Collateralized Lending

⚠️ **Diferença Crítica**: DeFi lending é sempre **over-collateralized** (colateral > empréstimo).

```solidity
// Exemplo Aave/Compound

1. User deposita 10 ETH (colateral) @ $2,000 = $20,000
2. Pode emprestar até 75% LTV (Loan-to-Value)
3. Max borrow: $15,000 em USDC
4. Se ETH cai para $1,600:
   - Colateral: $16,000
   - Empréstimo: $15,000
   - LTV: 93.75% ← Muito alto!
5. Liquidação: Protocolo vende parte do ETH para pagar dívida
```

### Implementação Simplificada

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract SimpleLending {
    IERC20 public immutable collateralToken; // Ex: WETH
    IERC20 public immutable borrowToken;      // Ex: USDC

    uint256 public constant LTV = 75; // 75% loan-to-value
    uint256 public constant LIQUIDATION_THRESHOLD = 80; // 80%
    uint256 public constant LIQUIDATION_PENALTY = 10; // 10%

    struct User {
        uint256 collateral;
        uint256 borrowed;
    }

    mapping(address => User) public users;

    uint256 public totalBorrows;
    uint256 public totalDeposits;

    // Preço oracle (simplificado - use Chainlink em produção!)
    uint256 public collateralPrice; // Em USD
    uint256 public borrowPrice = 1e18; // USDC = $1

    event Deposit(address indexed user, uint256 amount);
    event Borrow(address indexed user, uint256 amount);
    event Repay(address indexed user, uint256 amount);
    event Liquidate(address indexed liquidator, address indexed user, uint256 amount);

    constructor(address _collateralToken, address _borrowToken) {
        collateralToken = IERC20(_collateralToken);
        borrowToken = IERC20(_borrowToken);
    }

    // Depositar colateral
    function deposit(uint256 amount) external {
        collateralToken.transferFrom(msg.sender, address(this), amount);
        users[msg.sender].collateral += amount;
        emit Deposit(msg.sender, amount);
    }

    // Pegar emprestado
    function borrow(uint256 amount) external {
        User storage user = users[msg.sender];

        // Calcular max borrow
        uint256 collateralValue = (user.collateral * collateralPrice) / 1e18;
        uint256 maxBorrow = (collateralValue * LTV) / 100;

        require(user.borrowed + amount <= maxBorrow, "Exceeds borrow limit");

        user.borrowed += amount;
        totalBorrows += amount;

        borrowToken.transfer(msg.sender, amount);

        emit Borrow(msg.sender, amount);
    }

    // Pagar dívida
    function repay(uint256 amount) external {
        User storage user = users[msg.sender];

        require(amount <= user.borrowed, "Repay exceeds debt");

        borrowToken.transferFrom(msg.sender, address(this), amount);

        user.borrowed -= amount;
        totalBorrows -= amount;

        emit Repay(msg.sender, amount);
    }

    // Retirar colateral
    function withdraw(uint256 amount) external {
        User storage user = users[msg.sender];

        require(amount <= user.collateral, "Insufficient collateral");

        // Verificar se ainda tem colateral suficiente após withdraw
        uint256 remainingCollateral = user.collateral - amount;
        uint256 collateralValue = (remainingCollateral * collateralPrice) / 1e18;
        uint256 maxBorrow = (collateralValue * LTV) / 100;

        require(user.borrowed <= maxBorrow, "Would exceed borrow limit");

        user.collateral -= amount;
        collateralToken.transfer(msg.sender, amount);
    }

    // Liquidar posição insolvente
    function liquidate(address borrower) external {
        User storage user = users[borrower];

        // Calcular health factor
        uint256 collateralValue = (user.collateral * collateralPrice) / 1e18;
        uint256 borrowValue = (user.borrowed * borrowPrice) / 1e18;
        uint256 currentLTV = (borrowValue * 100) / collateralValue;

        require(currentLTV >= LIQUIDATION_THRESHOLD, "Position is healthy");

        // Liquidador paga dívida
        borrowToken.transferFrom(msg.sender, address(this), user.borrowed);

        // Liquidador recebe colateral + penalty
        uint256 collateralToSeize = user.collateral;
        uint256 bonus = (collateralToSeize * LIQUIDATION_PENALTY) / 100;

        collateralToken.transfer(msg.sender, collateralToSeize + bonus);

        totalBorrows -= user.borrowed;
        user.borrowed = 0;
        user.collateral = 0;

        emit Liquidate(msg.sender, borrower, user.borrowed);
    }

    // Admin: Atualizar preço (use Chainlink em produção!)
    function updatePrice(uint256 newPrice) external {
        collateralPrice = newPrice;
    }

    // View: Health factor do usuário
    function getHealthFactor(address user_) external view returns (uint256) {
        User storage user = users[user_];
        if (user.borrowed == 0) return type(uint256).max;

        uint256 collateralValue = (user.collateral * collateralPrice) / 1e18;
        uint256 borrowValue = (user.borrowed * borrowPrice) / 1e18;

        return (collateralValue * 100) / borrowValue;
    }
}
```

### Interest Rate Model

**Problema**: Como determinar taxa de juros?

**Solução**: Algorítmica baseada em **utilization rate**!

```solidity
contract InterestRateModel {
    // Utilization = Total Borrowed / Total Supplied
    function getUtilizationRate(uint256 borrowed, uint256 supplied) public pure returns (uint256) {
        if (supplied == 0) return 0;
        return (borrowed * 1e18) / supplied;
    }

    // Taxa aumenta conforme utilization
    function getBorrowRate(uint256 utilization) public pure returns (uint256) {
        // Modelo linear simples
        // 0% util = 2% APR
        // 100% util = 20% APR

        uint256 baseRate = 2e16; // 2%
        uint256 multiplier = 18e16; // 18%

        return baseRate + (utilization * multiplier) / 1e18;
    }

    // Supply APY = Borrow APR * Utilization * (1 - Reserve Factor)
    function getSupplyRate(uint256 utilization, uint256 borrowRate) public pure returns (uint256) {
        uint256 reserveFactor = 10; // 10%
        uint256 rateToPool = (borrowRate * (100 - reserveFactor)) / 100;
        return (utilization * rateToPool) / 1e18;
    }
}
```

---

## 🔍 Estudo de Caso: Aave V3 - Lending Protocol em Produção

### Aave Protocol

**Protocolo**: Aave V3
**TVL**: ~$20B+ (cross-chain)
**Endereço (Ethereum)**: [0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2](https://etherscan.io/address/0x87870bca3f3fd6335c3f4ce8392d69350b4fa4e2)
**Ativos suportados**: 30+ tokens
**Chains**: Ethereum, Polygon, Arbitrum, Optimism, Avalanche, Base

### Arquitetura: Pool-Based vs Isolated Markets

**Nossa implementação**: 1 contrato, 1 par de tokens

**Aave**: Pool único com múltiplos assets

```
SimpleLending:
- 1 contrato por par (WETH/USDC)
- Deploy novo para cada market
- Liquidez fragmentada

Aave:
- 1 Pool para TODOS os assets
- Compartilham liquidez
- Eficiência de capital máxima
- Cross-collateral (deposita ETH, pega DAI)
```

### Nossa Implementação vs Aave V3

**SimpleLending (Tutorial)**:
```solidity
contract SimpleLending {
    IERC20 public collateralToken;
    IERC20 public borrowToken;

    mapping(address => uint256) public collateral;
    mapping(address => uint256) public borrowed;
}
```

**Aave V3 (Produção)**:
```solidity
// CORE: Pool.sol (gerencia TODOS os assets)
contract Pool {
    // Cada asset tem sua configuração
    mapping(address => DataTypes.ReserveData) internal _reserves;
    mapping(uint256 => address) internal _reservesList;
    mapping(address => DataTypes.UserConfigurationMap) internal _usersConfig;

    // Supply (depositar)
    function supply(
        address asset,
        uint256 amount,
        address onBehalfOf,
        uint16 referralCode
    ) external {
        DataTypes.ReserveData storage reserve = _reserves[asset];

        // Valida reserve
        ValidationLogic.validateSupply(reserve, amount);

        // Transfer asset
        IERC20(asset).safeTransferFrom(msg.sender, reserve.aTokenAddress, amount);

        // Mint aTokens (interest-bearing tokens)
        IAToken(reserve.aTokenAddress).mint(
            onBehalfOf,
            amount,
            reserve.liquidityIndex
        );

        emit Supply(asset, msg.sender, onBehalfOf, amount, referralCode);
    }

    // Borrow
    function borrow(
        address asset,
        uint256 amount,
        uint256 interestRateMode, // 1 = Stable, 2 = Variable
        uint16 referralCode,
        address onBehalfOf
    ) external {
        DataTypes.ReserveData storage reserve = _reserves[asset];

        // Valida borrow
        (bool isActive, , , bool borrowingEnabled, bool stableBorrowEnabled) = reserve.getFlags();
        require(isActive && borrowingEnabled, "Reserve not active");

        // Check collateral & health factor
        (uint256 totalCollateralBase, uint256 totalDebtBase, , , uint256 healthFactor) =
            GenericLogic.calculateUserAccountData(/*...*/);

        require(healthFactor >= 1e18, "Health factor too low");

        // Update state
        reserve.updateState();

        // Update rates
        reserve.updateInterestRates(asset, reserve.aTokenAddress, 0, amount);

        // Mint debt tokens
        if (interestRateMode == 2) {
            IVariableDebtToken(reserve.variableDebtTokenAddress).mint(
                onBehalfOf,
                amount,
                reserve.variableBorrowIndex
            );
        }

        // Transfer asset to user
        IAToken(reserve.aTokenAddress).transferUnderlyingTo(msg.sender, amount);

        emit Borrow(asset, msg.sender, onBehalfOf, amount, interestRateMode, reserve.currentVariableBorrowRate, referralCode);
    }
}
```

### Diferenças-Chave: Tutorial → Produção

| Aspecto | SimpleLending | Aave V3 | Por Quê |
|---------|---------------|---------|---------|
| **Arquitetura** | 1 par de tokens | Pool com 30+ assets | Eficiência de capital, cross-collateral |
| **Juros** | Fixo ou simples | Compound interest | Precisão, incentivos dinâmicos |
| **Collateral** | Único tipo | Múltiplos assets | Flexibilidade (ETH + LINK + UNI) |
| **Interest Tokens** | Nenhum | aTokens (ERC-20) | Composability, yield auto-compounding |
| **Debt Tokens** | Mapping interno | debtTokens (ERC-20) | Transferível, integração DeFi |
| **Taxas** | Simples (1 rate) | Stable vs Variable | Choice para borrowers |
| **Isolation Mode** | ❌ Não | ✅ Sim | Protege protocolo de assets arriscados |
| **E-Mode** | ❌ Não | ✅ Efficiency mode | LTV 97% para stablecoins correlacionadas |
| **Flash Loans** | ❌ Não | ✅ Nativo | Revenue + utilidade |
| **Liquidations** | All-or-nothing | Parcial + Dutch auction | Menos disruptivo, mais eficiente |

### aTokens - Interest-Bearing Tokens

**Inovação-chave do Aave**: Seu depósito é um token ERC-20!

```solidity
// Você deposita 100 USDC
aave.supply(USDC, 100e6, msg.sender, 0);

// Recebe 100 aUSDC (aToken)
// aUSDC.balanceOf(msg.sender) = 100e6

// Após 1 ano (assumindo 5% APY)
// aUSDC.balanceOf(msg.sender) = 105e6 (cresce automaticamente!)

// aTokens são ERC-20, então:
- Transferível
- Usável como colateral em outros protocolos
- Composable (yield-bearing collateral)
```

**Implementação de aToken**:
```solidity
contract AToken is ERC20 {
    // Balance cresce com juros automaticamente
    function balanceOf(address user) public view override returns (uint256) {
        return super.balanceOf(user).rayMul(pool.getReserveNormalizedIncome(underlyingAsset));
    }

    // Scaled balance (sem juros acumulados)
    function scaledBalanceOf(address user) public view returns (uint256) {
        return super.balanceOf(user);
    }

    // Mint com scaling
    function mint(
        address user,
        uint256 amount,
        uint256 index
    ) external onlyPool returns (bool) {
        uint256 amountScaled = amount.rayDiv(index);
        _mint(user, amountScaled);
        return true;
    }
}
```

**Por que scaling?** Evita atualizar saldo de TODOS users a cada bloco!

### Debt Tokens - Dívida Como ERC-20

```solidity
// Stable Debt Token (taxa fixa)
contract StableDebtToken is ERC20 {
    // Não transferível (soulbound)
    function transfer(address, uint256) public override returns (bool) {
        revert("TRANSFER_NOT_SUPPORTED");
    }

    // Balance cresce com juros
    function balanceOf(address user) public view override returns (uint256) {
        uint256 accountBalance = super.balanceOf(user);
        uint256 stableRate = _userStableRates[user];
        return accountBalance.rayMul(_calculateCompoundedInterest(stableRate, _timestamps[user]));
    }
}

// Variable Debt Token (taxa variável)
contract VariableDebtToken is ERC20 {
    // Similar, mas usa taxa variável do pool
    function balanceOf(address user) public view override returns (uint256) {
        return super.balanceOf(user).rayMul(pool.getReserveNormalizedVariableDebt(underlyingAsset));
    }
}
```

**Vantagem**: Debt position é visível on-chain como token!

### Health Factor Avançado

**Nossa versão**: LTV simples

**Aave**: Health factor considerando TODOS colaterais e dívidas

```solidity
function calculateUserAccountData(address user)
    returns (
        uint256 totalCollateralBase,
        uint256 totalDebtBase,
        uint256 availableBorrowsBase,
        uint256 currentLiquidationThreshold,
        uint256 ltv,
        uint256 healthFactor
    )
{
    // Itera todos os assets que user tem
    for (uint256 i = 0; i < reservesCount; i++) {
        if (userConfig.isUsingAsCollateral(i)) {
            (uint256 balance, , , , ) = reserve.aTokenAddress.balanceOf(user);
            uint256 valueInBase = balance * assetPrice;

            totalCollateralBase += valueInBase * reserve.ltv;
            currentLiquidationThreshold += valueInBase * reserve.liquidationThreshold;
        }

        if (userConfig.isBorrowing(i)) {
            uint256 debt = reserve.debtToken.balanceOf(user);
            totalDebtBase += debt * assetPrice;
        }
    }

    // Health factor = (Collateral * Liquidation Threshold) / Total Debt
    healthFactor = (totalCollateralBase * currentLiquidationThreshold) / totalDebtBase;

    // healthFactor < 1e18 = liquidatable
}
```

### E-Mode (Efficiency Mode)

**Problema**: ETH e stETH são quase 1:1, mas LTV é 75%

**Solução Aave V3**: E-Mode com LTV até 97%!

```solidity
// Definir categorias de assets correlacionados
eModeCategoryData[1] = {
    ltv: 9700,  // 97%
    liquidationThreshold: 9800,  // 98%
    liquidationBonus: 10100,  // 1% bonus
    priceSource: address(0),  // Use oracle padrão
    label: "ETH correlated"
};

// User entra em E-Mode para categoria ETH
pool.setUserEMode(1);

// Agora pode emprestar 97 ETH contra 100 stETH!
```

**Trade-off**: Só pode pegar emprestado assets da mesma categoria.

### Isolation Mode

**Problema**: Como listar assets novos sem arriscar protocolo inteiro?

**Solução Aave V3**: Isolation mode!

```solidity
// Asset novo (ex: SHIB) marcado como "isolated"
reserve.setIsolated(true);
reserve.setDebtCeiling(1_000_000e18); // Max $1M debt

// Se user usa SHIB como collateral:
- Só pode pegar emprestado stablecoins
- Limited to debt ceiling
- Protege protocolo de assets voláteis
```

### Flash Loans Nativos

**Nossa versão**: Não suporta

**Aave**: Flash loans como feature core

```solidity
contract Pool {
    function flashLoan(
        address receiverAddress,
        address[] calldata assets,
        uint256[] calldata amounts,
        uint256[] calldata modes, // 0 = no debt, 1 = stable, 2 = variable
        address onBehalfOf,
        bytes calldata params,
        uint16 referralCode
    ) external {
        // 1. Valida
        // 2. Transfer assets to receiver
        for (uint256 i = 0; i < assets.length; i++) {
            IAToken(reserves[assets[i]].aTokenAddress).transferUnderlyingTo(receiverAddress, amounts[i]);
        }

        // 3. Callback
        require(
            IFlashLoanReceiver(receiverAddress).executeOperation(
                assets,
                amounts,
                premiums, // 0.09% fee
                msg.sender,
                params
            ),
            "Flash loan callback failed"
        );

        // 4. Verifica repayment (ou abre dívida se mode != 0)
        for (uint256 i = 0; i < assets.length; i++) {
            if (modes[i] == 0) {
                // Repay + premium
                require(
                    IERC20(assets[i]).balanceOf(aTokenAddress) >= amountPlusPremium,
                    "Flash loan not repaid"
                );
            } else {
                // Abre dívida (flash loan vira empréstimo!)
                _executeBorrow(/*...*/);
            }
        }
    }
}
```

**Revenue do protocolo**: 0.09% fee em bilhões de volume = milhões em fees!

### Liquidações Avançadas

**Nossa versão**: Liquidate tudo

**Aave**: Liquidações parciais + Dutch auction

```solidity
function liquidationCall(
    address collateralAsset,
    address debtAsset,
    address user,
    uint256 debtToCover,
    bool receiveAToken
) external {
    // 1. Valida health factor < 1
    require(healthFactor < 1e18, "Position is healthy");

    // 2. Limite de liquidação parcial (50% max)
    uint256 maxLiquidatableDebt = (userDebt * CLOSE_FACTOR) / 10000; // 50%
    uint256 actualDebtToLiquidate = min(debtToCover, maxLiquidatableDebt);

    // 3. Bonus para liquidator (5-10%)
    uint256 bonusCollateral = actualDebtToLiquidate * (10000 + liquidationBonus) / 10000;

    // 4. Transfer debt from liquidator
    IERC20(debtAsset).transferFrom(msg.sender, aTokenAddress, actualDebtToLiquidate);

    // 5. Transfer collateral to liquidator
    if (receiveAToken) {
        // Recebe aToken (mantém yield)
        aToken.transferFrom(user, msg.sender, bonusCollateral);
    } else {
        // Recebe underlying asset
        aToken.burn(user, bonusCollateral);
        IERC20(collateralAsset).transfer(msg.sender, bonusCollateral);
    }

    emit LiquidationCall(/*...*/);
}
```

**Vantagens**:
- Parcial: Não liquida tudo de uma vez
- Bonus dinâmico: Aumenta se health factor muito baixo
- receiveAToken option: Liquidator pode ficar com yield

### Interest Rate Strategy

**Nossa versão**: Linear simples

**Aave**: Kinked rate model (2 slopes)

```solidity
contract DefaultReserveInterestRateStrategy {
    // Model com 2 slopes (kink)
    uint256 immutable OPTIMAL_UTILIZATION_RATE; // Ex: 80%
    uint256 immutable BASE_VARIABLE_BORROW_RATE; // 0%
    uint256 immutable VARIABLE_RATE_SLOPE1; // Slope até optimal
    uint256 immutable VARIABLE_RATE_SLOPE2; // Slope após optimal

    function calculateInterestRates(
        uint256 liquidityAdded,
        uint256 liquidityTaken,
        uint256 totalVariableDebt,
        uint256 reserveFactor
    ) external view returns (uint256, uint256) {
        uint256 totalDebt = totalVariableDebt;
        uint256 totalLiquidity = liquidityAdded + totalVariableDebt;

        uint256 utilizationRate = totalDebt / totalLiquidity;

        uint256 variableBorrowRate;

        if (utilizationRate <= OPTIMAL_UTILIZATION_RATE) {
            // Slope suave (0-80% utilization)
            variableBorrowRate = BASE_VARIABLE_BORROW_RATE +
                (utilizationRate * VARIABLE_RATE_SLOPE1) / OPTIMAL_UTILIZATION_RATE;
        } else {
            // Slope íngreme (80-100% utilization)
            uint256 excessUtil = utilizationRate - OPTIMAL_UTILIZATION_RATE;
            variableBorrowRate = BASE_VARIABLE_BORROW_RATE +
                VARIABLE_RATE_SLOPE1 +
                (excessUtil * VARIABLE_RATE_SLOPE2) / (1e27 - OPTIMAL_UTILIZATION_RATE);
        }

        uint256 supplyRate = (variableBorrowRate * utilizationRate * (10000 - reserveFactor)) / 10000;

        return (supplyRate, variableBorrowRate);
    }
}
```

**Exemplo**:
```
0-80% utilization: 0% → 4% (gradual)
80-100% utilization: 4% → 80% (íngreme!) ← Incentiva repayments
```

### Lições para Produção

1. **Tokens para positions**: aTokens e debtTokens permitem composability
2. **Scaling vs Balance updates**: Evita atualizar milhares de users
3. **Multi-asset pool**: Compartilha liquidez, cross-collateral
4. **E-Mode**: High LTV para assets correlacionados (stablecoins, ETH derivatives)
5. **Isolation mode**: Liste assets novos com risco controlado
6. **Flash loans**: Revenue stream + útil pro ecossistema
7. **Partial liquidations**: Menos disruptivo para borrowers
8. **Kinked interest model**: Incentiva utilization ideal
9. **Modular design**: ReserveLogic, ValidationLogic, LiquidationLogic separados
10. **Upgradeable via proxy**: Bug fixes sem migração completa

### Comparação: Aave vs Compound

| Feature | Aave | Compound | Quando Usar |
|---------|------|----------|-------------|
| **Interest tokens** | aTokens (1:1 + yield) | cTokens (compounding ratio) | Aave mais intuitivo |
| **Collateral** | Toggle per asset | Automatic | Aave mais flexível |
| **Flash loans** | ✅ Nativo | ❌ Não | Aave para arbitrage |
| **E-Mode** | ✅ Sim | ❌ Não | Aave para stablecoins |
| **Stable rates** | ✅ Sim | ❌ Só variable | Aave para previsibilidade |
| **Governance** | AAVE token | COMP token | Similar |
| **TVL** | ~$20B | ~$3B | Aave domina atualmente |

---

## 10.6 Staking Contracts

### Staking Básico

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract SimpleStaking {
    IERC20 public immutable stakingToken;
    IERC20 public immutable rewardToken;

    uint256 public rewardRate = 100; // 100 tokens por segundo
    uint256 public lastUpdateTime;
    uint256 public rewardPerTokenStored;

    mapping(address => uint256) public userRewardPerTokenPaid;
    mapping(address => uint256) public rewards;

    uint256 public totalStaked;
    mapping(address => uint256) public balances;

    constructor(address _stakingToken, address _rewardToken) {
        stakingToken = IERC20(_stakingToken);
        rewardToken = IERC20(_rewardToken);
        lastUpdateTime = block.timestamp;
    }

    // Calcular reward por token
    function rewardPerToken() public view returns (uint256) {
        if (totalStaked == 0) return rewardPerTokenStored;

        return rewardPerTokenStored +
            ((block.timestamp - lastUpdateTime) * rewardRate * 1e18) / totalStaked;
    }

    // Calcular rewards pendentes do usuário
    function earned(address account) public view returns (uint256) {
        return (balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account])) / 1e18
            + rewards[account];
    }

    // Modifier para atualizar rewards
    modifier updateReward(address account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;

        if (account != address(0)) {
            rewards[account] = earned(account);
            userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }

    // Stake
    function stake(uint256 amount) external updateReward(msg.sender) {
        require(amount > 0, "Cannot stake 0");

        totalStaked += amount;
        balances[msg.sender] += amount;

        stakingToken.transferFrom(msg.sender, address(this), amount);
    }

    // Unstake
    function withdraw(uint256 amount) external updateReward(msg.sender) {
        require(amount > 0, "Cannot withdraw 0");

        totalStaked -= amount;
        balances[msg.sender] -= amount;

        stakingToken.transfer(msg.sender, amount);
    }

    // Claim rewards
    function getReward() external updateReward(msg.sender) {
        uint256 reward = rewards[msg.sender];
        if (reward > 0) {
            rewards[msg.sender] = 0;
            rewardToken.transfer(msg.sender, reward);
        }
    }

    // Exit (unstake + claim)
    function exit() external {
        withdraw(balances[msg.sender]);
        getReward();
    }
}
```

### Yield Farming com LP Tokens

```solidity
contract YieldFarm {
    // Stake LP tokens, earn governance token
    IERC20 public lpToken; // LP token do DEX
    IERC20 public rewardToken; // Governance token

    // Múltiplos pools com weights diferentes
    struct PoolInfo {
        IERC20 lpToken;
        uint256 allocPoint; // Peso do pool
        uint256 lastRewardBlock;
        uint256 accRewardPerShare;
    }

    PoolInfo[] public poolInfo;
    mapping(uint256 => mapping(address => uint256)) public userStaked;
    mapping(uint256 => mapping(address => uint256)) public rewardDebt;

    uint256 public rewardPerBlock = 10 * 1e18; // 10 tokens por bloco
    uint256 public totalAllocPoint = 0;

    // Adicionar pool
    function add(uint256 _allocPoint, IERC20 _lpToken) external {
        poolInfo.push(PoolInfo({
            lpToken: _lpToken,
            allocPoint: _allocPoint,
            lastRewardBlock: block.number,
            accRewardPerShare: 0
        }));
        totalAllocPoint += _allocPoint;
    }

    // Atualizar pool
    function updatePool(uint256 pid) public {
        PoolInfo storage pool = poolInfo[pid];

        if (block.number <= pool.lastRewardBlock) return;

        uint256 lpSupply = pool.lpToken.balanceOf(address(this));
        if (lpSupply == 0) {
            pool.lastRewardBlock = block.number;
            return;
        }

        uint256 blocks = block.number - pool.lastRewardBlock;
        uint256 reward = (blocks * rewardPerBlock * pool.allocPoint) / totalAllocPoint;

        pool.accRewardPerShare += (reward * 1e12) / lpSupply;
        pool.lastRewardBlock = block.number;
    }

    // Stake
    function stake(uint256 pid, uint256 amount) external {
        PoolInfo storage pool = poolInfo[pid];
        updatePool(pid);

        if (userStaked[pid][msg.sender] > 0) {
            uint256 pending = (userStaked[pid][msg.sender] * pool.accRewardPerShare) / 1e12
                - rewardDebt[pid][msg.sender];
            rewardToken.transfer(msg.sender, pending);
        }

        pool.lpToken.transferFrom(msg.sender, address(this), amount);
        userStaked[pid][msg.sender] += amount;
        rewardDebt[pid][msg.sender] = (userStaked[pid][msg.sender] * pool.accRewardPerShare) / 1e12;
    }
}
```

---

## 10.7 Security em DeFi

### Vulnerabilidades Comuns

#### 1. Price Oracle Manipulation

```solidity
// ❌ VULNERÁVEL: Usar preço spot do próprio DEX
function getPrice() public view returns (uint256) {
    return (reserve1 * 1e18) / reserve0; // Pode ser manipulado com flash loan!
}

// ✅ CORRETO: Usar TWAP (Time-Weighted Average Price) ou Chainlink
function getPrice() public view returns (uint256) {
    // Uniswap V2 TWAP
    return IUniswapV2Pair(pair).price0CumulativeLast();

    // Ou Chainlink
    return IPriceFeed(oracle).latestAnswer();
}
```

#### 2. Flash Loan Attacks

```solidity
// Cenário de ataque:
// 1. Flash loan: Pegar 1M USDC emprestado
// 2. Comprar muito ETH no DEX vulnerável
// 3. Preço do ETH sobe artificialmente
// 4. Oracle do lending protocol lê preço manipulado
// 5. Emprestar contra colateral inflado
// 6. Repagar flash loan
// 7. Profit!

// Proteção: TWAP, circuit breakers, limites
```

#### 3. Reentrancy em DeFi

```solidity
// ❌ VULNERÁVEL
function removeLiquidity(uint256 amount) external {
    uint256 eth = calculateETH(amount);

    (bool success, ) = msg.sender.call{value: eth}(""); // Reentrancy aqui!
    require(success);

    _burn(msg.sender, amount); // State update tarde demais
}

// ✅ CORRETO: Checks-Effects-Interactions
function removeLiquidity(uint256 amount) external {
    uint256 eth = calculateETH(amount);

    _burn(msg.sender, amount); // State update primeiro

    (bool success, ) = msg.sender.call{value: eth}("");
    require(success);
}
```

#### 4. Integer Overflow em Cálculos Financeiros

```solidity
// ❌ PERIGOSO mesmo com Solidity 0.8+
function calculateReward(uint256 balance, uint256 rate, uint256 time) public pure returns (uint256) {
    return (balance * rate * time) / 1e18; // Pode overflow se valores grandes!
}

// ✅ SEGURO: Usar ordem correta
function calculateReward(uint256 balance, uint256 rate, uint256 time) public pure returns (uint256) {
    return (balance * rate) / 1e18 * time; // Divide antes de multiplicar
}
```

#### 5. Slippage e Front-Running

```solidity
// ❌ SEM PROTEÇÃO
function swap(uint256 amountIn) external {
    uint256 amountOut = getAmountOut(amountIn);
    // Front-runner pode fazer trade grande antes, piorando seu preço
    _swap(amountIn, amountOut);
}

// ✅ COM SLIPPAGE PROTECTION
function swap(uint256 amountIn, uint256 minAmountOut) external {
    uint256 amountOut = getAmountOut(amountIn);
    require(amountOut >= minAmountOut, "Slippage too high");
    _swap(amountIn, amountOut);
}
```

---

## 📖 Glossário

**AMM (Automated Market Maker)**
> Protocolo que usa fórmula matemática para determinar preços, em vez de order book.
> **Analogia**: Como vending machine (preço fixo) vs mercado tradicional (negociação).
> **Por que existe**: Order books são caros on-chain (cada ordem = gas).

**Liquidity Pool**
> Reserva de tokens que permite swaps instantâneos.
> **Analogia**: Como buffer/cache em sistemas - sempre tem liquidez disponível.

**Impermanent Loss**
> Perda (temporária) de valor ao prover liquidez vs simplesmente segurar tokens.
> **Por que acontece**: Rebalanceamento do pool via arbitrage quando preços mudam.

**LTV (Loan-to-Value)**
> Razão entre valor emprestado e valor do colateral.
> **Exemplo**: $7,500 emprestado / $10,000 colateral = 75% LTV.

**Liquidation**
> Venda forçada de colateral quando LTV ultrapassa threshold.
> **Por que existe**: Proteger protocolo de bad debt.

**Flash Loan**
> Empréstimo sem colateral que deve ser pago na mesma transação.
> **Uso legítimo**: Arbitrage, refinanciamento.
> **Uso malicioso**: Manipular oracles, ataques de governança.

**TWAP (Time-Weighted Average Price)**
> Preço médio ponderado por tempo, resistente a manipulação.
> **Analogia**: Como média móvel em análise técnica.

**Utilization Rate**
> Percentual do capital disponível que está emprestado.
> **Fórmula**: Total Borrowed / Total Supplied.

**Yield Farming**
> Estratégia de maximizar retorno movendo capital entre protocolos.
> **Analogia**: Como arbitrage de juros em finanças tradicionais.

---

## 🔒 Security Checklist: DeFi

### Smart Contract
- [ ] Usar oracles resistentes a manipulação (Chainlink, TWAP)
- [ ] Checks-Effects-Interactions em todas transfers
- [ ] Slippage protection em swaps
- [ ] Reentrancy guards em funções públicas
- [ ] Pausable para emergências
- [ ] Timelock para mudanças críticas

### Matemática e Cálculos
- [ ] Verificar ordem de operações (evitar overflow)
- [ ] Usar library de math seguro (PRBMath, OpenZeppelin)
- [ ] Testar edge cases (zero, max uint, etc.)
- [ ] Precisão adequada (não perder decimais)

### Oracles e Preços
- [ ] NUNCA usar preço spot de único DEX
- [ ] Implementar TWAP ou usar Chainlink
- [ ] Circuit breakers para mudanças abruptas
- [ ] Multiple oracles como backup

### Liquidações
- [ ] Incentivo suficiente para liquidators
- [ ] Grace period antes de liquidação
- [ ] Partial liquidations (não tudo de uma vez)
- [ ] Proteção contra liquidação em cascata

### Testes
- [ ] Fork tests contra protocolos reais (Uniswap, Aave)
- [ ] Fuzzing de cálculos matemáticos
- [ ] Simular ataques conhecidos
- [ ] Testar com preços extremos

---

## 📝 Exercícios Práticos

### Exercício 1: Build Your Own AMM

**Objetivo**: Implementar DEX funcional com constant product formula.

**Descrição**: Criar contrato que:
1. Aceite dois tokens ERC-20
2. Permita adicionar/remover liquidez
3. Execute swaps com 0.3% fee
4. Calcule price impact
5. Emita LP tokens

**Requisitos**:
- Slippage protection
- Mínimo de liquidez (1000 wei) para prevenir manipulação
- Events para todas operações
- View functions para simular swaps

<details>
<summary>💡 Dica</summary>

Para calcular LP tokens na primeira adição de liquidez, use `sqrt(amount0 * amount1)` (similar ao Uniswap V2). Isso previne manipulação de preço inicial.

Para swaps, lembre que `(x + dx) * (y - dy) = k`, então `dy = y * dx / (x + dx)`.
</details>

<details>
<summary>✅ Solução</summary>

Veja implementação completa em `SimpleDEX` na seção 10.4. Pontos críticos:

1. **LP Tokens**: Use `sqrt` na primeira liquidez para prevenir front-running do primeiro LP
2. **Fee**: Aplicar antes de calcular output: `amountInWithFee = amountIn * 997`
3. **Slippage**: Sempre aceitar `minAmountOut` para proteger usuário
4. **Updates**: Atualizar reserves APÓS todas transfers

</details>

---

### Exercício 2: Lending com Liquidações

**Objetivo**: Implementar lending protocol com sistema de liquidação.

**Descrição**:
- Permitir deposit de colateral (WETH)
- Emprestar stablecoin (USDC)
- Implementar health factor
- Permitir liquidações quando health < 1

**Desafio Extra**:
- Implementar interest accrual (juros compostos)
- Adicionar múltiplos tipos de colateral
- Criar incentivo para liquidators (liquidation bonus)

<details>
<summary>💡 Dica</summary>

Health factor: `(Collateral Value * Liquidation Threshold) / Borrow Value`

Se health < 1, posição é liquidável.

Para juros, use: `debt = principal * (1 + rate)^time`

Mas em Solidity, calcule incrementalmente a cada bloco para evitar overflow.
</details>

---

### Exercício 3: Flash Loan Attack Simulation

**Objetivo**: Entender vetores de ataque via flash loans.

**Descrição**:
Dado um DEX vulnerável que usa seu próprio preço spot como oracle:
1. Implementar flash loan provider
2. Criar contrato de ataque
3. Manipular preço e extrair valor

**Aprendizado**: Ver POR QUE oracles centralizados são perigosos.

⚠️ **Ética**: Este exercício é educacional. NUNCA use em mainnet.

<details>
<summary>💡 Estrutura do Ataque</summary>

```solidity
// 1. Flash loan 1M USDC
// 2. Swap 1M USDC → ETH (preço sobe)
// 3. Oracle vulnerável lê preço inflado
// 4. Usar preço falso para lucrar (ex: lending)
// 5. Repagar flash loan
// 6. Profit = diferença
```

**Proteção**: TWAP, Chainlink, circuit breakers.
</details>

---

## 📚 Recursos Adicionais

### Documentação Essencial

1. **[Uniswap V2 Whitepaper](https://uniswap.org/whitepaper.pdf)**
   - Matemática completa do constant product AMM
   - Referência definitiva para entender x * y = k

2. **[Aave Documentation](https://docs.aave.com/)**
   - Lending protocol mais usado
   - Interest rate models, liquidations, flash loans

3. **[Compound Documentation](https://compound.finance/docs)**
   - cToken model (alternativa para LP tokens)
   - Governance com COMP token

### Ferramentas

- **[DeFi Llama](https://defillama.com/)** - TVL tracking, compare protocolos
- **[Tenderly](https://tenderly.co/)** - Simulate transações, debug DeFi
- **[Uniswap SDK](https://docs.uniswap.org/sdk/v3/overview)** - Integrar com Uniswap

### Deep Dives

- **[Finematics YouTube](https://www.youtube.com/c/Finematics)** - Visualizações de conceitos DeFi
- **[Smart Contract Programmer - DeFi Series](https://www.youtube.com/watch?v=qB2Ulx201wY)** - Código linha por linha
- **[Rekt News](https://rekt.news/)** - Post-mortems de hacks DeFi (aprenda com erros dos outros)

---

## 🎯 Próximos Passos

**Você agora entende**:
- ✅ Como AMMs funcionam matematicamente
- ✅ Por que impermanent loss acontece
- ✅ Lending over-collateralized
- ✅ Liquidações e health factors
- ✅ Staking e yield farming
- ✅ Principais vetores de ataque em DeFi

**Próximo Capítulo**: [Capítulo 11 - Oracles](./EBOOK_CAPITULO_11_ORACLES.md)
- Como trazer dados off-chain de forma segura
- Chainlink Price Feeds, VRF, Automation
- Oracle problem e soluções
- Construir custom oracle (quando faz sentido)

**Aplicação Prática**:
Agora você pode construir protocolos DeFi reais! Combine:
- Cap 9 (Tokens) → Criar governance token
- Cap 10 (DeFi) → Build DEX + Staking
- Cap 11 (Oracles) → Adicionar price feeds confiáveis
- Cap 8 (Security) → Auditar tudo antes de deploy

💡 **Projeto Sugerido**: Clone simplificado de Uniswap + staking de LP tokens para ganhar governance token. Use Chainlink para tracking de preços. Deploy em testnet e adicione UI (Cap 13).

---

**Autor**: Baseado em roadmap ITA Blockchain Club + análise de protocolos reais (Uniswap V2, Aave, Compound)
**Última Atualização**: 2025-01-17
**Changelog**: Adicionados estudos de caso detalhados de Uniswap V2 e Aave V3 com análises de arquitetura, otimizações e diferenças entre implementações tutorial vs produção
