# Capítulo 11: Oracles e Dados Off-Chain

> **Para Desenvolvedores Experientes**: Se você trabalha com APIs REST, sabe que dados externos são triviais - uma chamada HTTP resolve. Em blockchain, isso é o "Oracle Problem": smart contracts não podem fazer HTTP requests. É como ter um servidor sem acesso à internet. Oracles são a ponte, mas implementar errado pode custar milhões.

---

## Índice
- [11.1 O Oracle Problem](#111-o-oracle-problem)
- [11.2 Chainlink - O Padrão de Mercado](#112-chainlink---o-padrão-de-mercado)
- [11.3 Price Feeds](#113-price-feeds)
- [11.4 Chainlink VRF - Randomness](#114-chainlink-vrf---randomness)
- [11.5 Chainlink Automation](#115-chainlink-automation)
- [11.6 Custom Oracles](#116-custom-oracles)
- [11.7 Oracle Attacks](#117-oracle-attacks)

---

## 11.1 O Oracle Problem

### Por Que Smart Contracts São "Cegos"

**Limitação Fundamental**:
```solidity
// ❌ IMPOSSÍVEL em smart contract
function getETHPrice() public returns (uint256) {
    // Não pode fazer HTTP request!
    return fetch("https://api.coinbase.com/v2/prices/ETH-USD/spot");
}
```

**Por que essa limitação existe?**

```
Blockchain = Determinístico
├─ Todos nodes devem chegar ao mesmo resultado
├─ HTTP request = não-determinístico (pode retornar valores diferentes)
└─ Se nodes divergem = blockchain quebra (fork)

Exemplo:
- Node A chama API às 10:00:00 → ETH = $2000
- Node B chama API às 10:00:01 → ETH = $2001
- State diverge → consenso quebra!
```

### Comparação Web2 vs Web3

| Aspecto | Web2 (Servidor Node.js) | Web3 (Smart Contract) |
|---------|------------------------|----------------------|
| **HTTP Requests** | ✅ Trivial (`fetch`, `axios`) | ❌ Impossível |
| **External APIs** | ✅ Qualquer API | ❌ Nenhuma diretamente |
| **Randomness** | ✅ `Math.random()` | ❌ Tudo é determinístico |
| **Cron Jobs** | ✅ `setInterval`, cron | ❌ Não há "tempo" automático |
| **File System** | ✅ `fs.readFile()` | ❌ Não há filesystem |

**Conclusão**: Smart contracts vivem em "caixa fechada" - só podem reagir a transações.

---

## 11.2 Chainlink - O Padrão de Mercado

### O Que É Chainlink

**Analogia Web2**: Chainlink é como API Gateway + CDN para blockchain.

```
Traditional API:
Frontend → HTTP → Backend API → Database

Blockchain Oracle:
Smart Contract → On-chain Request → Oracle Network → External Data → Response On-chain
```

### Arquitetura Chainlink

```
┌─────────────────────────────────────────┐
│        Smart Contract (On-chain)        │
│  ┌──────────────────────────────────┐   │
│  │ requestETHPrice()                │   │
│  │ fulfillPrice(uint256 price)      │   │
│  └──────────────────────────────────┘   │
└─────────────┬───────────────────────────┘
              │ 1. Request
              ▼
┌─────────────────────────────────────────┐
│     Chainlink Oracle Network            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ Node 1  │  │ Node 2  │  │ Node 3  │ │
│  └────┬────┘  └────┬────┘  └────┬────┘ │
└───────┼───────────┼─────────────┼───────┘
        │ 2. Fetch  │             │
        ▼           ▼             ▼
┌─────────────────────────────────────────┐
│      External Data Sources              │
│  (Coinbase, Binance, Kraken, etc.)      │
└─────────────────────────────────────────┘
        │ 3. Aggregate
        ▼
┌─────────────────────────────────────────┐
│      Median Price = $2,000              │
│  (Node 1: $2001, Node 2: $2000, Node 3: $1999) │
└─────────────────────────────────────────┘
        │ 4. Response
        ▼
┌─────────────────────────────────────────┐
│  Smart Contract recebe: $2,000          │
└─────────────────────────────────────────┘
```

### Por Que Chainlink É Confiável

1. **Descentralizado**: Múltiplos nodes independentes
2. **Agregação**: Mediana de múltiplas fontes
3. **Reputação**: Nodes têm stake, perdem se mentir
4. **Transparente**: Todas respostas on-chain, auditáveis

---

## 11.3 Price Feeds

### Usando Chainlink Price Feeds

**Caso de uso**: DeFi precisa saber preço de assets (ETH, BTC, etc.)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PriceConsumer {
    AggregatorV3Interface internal priceFeed;

    /**
     * Network: Sepolia
     * Aggregator: ETH/USD
     * Address: 0x694AA1769357215DE4FAC081bf1f309aDC325306
     */
    constructor() {
        priceFeed = AggregatorV3Interface(
            0x694AA1769357215DE4FAC081bf1f309aDC325306
        );
    }

    /**
     * Retorna último preço de ETH/USD
     */
    function getLatestPrice() public view returns (int) {
        (
            /* uint80 roundID */,
            int price,
            /* uint startedAt */,
            /* uint timeStamp */,
            /* uint80 answeredInRound */
        ) = priceFeed.latestRoundData();

        return price; // Preço com 8 decimais (ex: 200000000000 = $2000.00)
    }

    /**
     * Retorna número de decimais
     */
    function getDecimals() public view returns (uint8) {
        return priceFeed.decimals(); // 8
    }

    /**
     * Retorna preço formatado
     */
    function getETHPriceInUSD() public view returns (uint256) {
        int price = getLatestPrice();
        uint8 decimals = getDecimals();

        // Converter para formato com 18 decimais (padrão Ethereum)
        return uint256(price) * 10 ** (18 - decimals);
    }

    /**
     * Converter ETH para USD
     */
    function convertETHtoUSD(uint256 ethAmount) public view returns (uint256) {
        uint256 ethPrice = getETHPriceInUSD(); // 18 decimals
        return (ethAmount * ethPrice) / 1e18;
    }

    /**
     * Exemplo: Lending protocol precisa saber valor do colateral
     */
    function getCollateralValue(uint256 ethCollateral) public view returns (uint256 usdValue) {
        return convertETHtoUSD(ethCollateral);
    }
}
```

### Aplicação Prática: Lending com Chainlink

```solidity
contract ChainlinkLending {
    AggregatorV3Interface internal ethPriceFeed;
    AggregatorV3Interface internal btcPriceFeed;

    IERC20 public weth;
    IERC20 public wbtc;
    IERC20 public usdc;

    struct Position {
        uint256 wethCollateral;
        uint256 wbtcCollateral;
        uint256 usdcBorrowed;
    }

    mapping(address => Position) public positions;

    uint256 public constant LTV = 75; // 75%
    uint256 public constant LIQUIDATION_THRESHOLD = 80; // 80%

    constructor(address _ethFeed, address _btcFeed) {
        ethPriceFeed = AggregatorV3Interface(_ethFeed);
        btcPriceFeed = AggregatorV3Interface(_btcFeed);
    }

    /**
     * Calcular valor total do colateral em USD
     */
    function getCollateralValueUSD(address user) public view returns (uint256) {
        Position storage pos = positions[user];

        // ETH collateral
        (, int ethPrice,,,) = ethPriceFeed.latestRoundData();
        uint256 ethValue = (pos.wethCollateral * uint256(ethPrice)) / 1e8;

        // BTC collateral
        (, int btcPrice,,,) = btcPriceFeed.latestRoundData();
        uint256 btcValue = (pos.wbtcCollateral * uint256(btcPrice)) / 1e8;

        return ethValue + btcValue;
    }

    /**
     * Health factor (deve ser > 100)
     */
    function getHealthFactor(address user) public view returns (uint256) {
        uint256 collateralValue = getCollateralValueUSD(user);
        uint256 borrowed = positions[user].usdcBorrowed;

        if (borrowed == 0) return type(uint256).max;

        return (collateralValue * 100) / borrowed;
    }

    /**
     * Emprestar USDC
     */
    function borrow(uint256 usdcAmount) external {
        uint256 collateralValue = getCollateralValueUSD(msg.sender);
        uint256 maxBorrow = (collateralValue * LTV) / 100;

        require(
            positions[msg.sender].usdcBorrowed + usdcAmount <= maxBorrow,
            "Exceeds borrow limit"
        );

        positions[msg.sender].usdcBorrowed += usdcAmount;
        usdc.transfer(msg.sender, usdcAmount);
    }

    /**
     * Liquidar se health factor < threshold
     */
    function liquidate(address borrower) external {
        uint256 healthFactor = getHealthFactor(borrower);
        require(healthFactor < LIQUIDATION_THRESHOLD, "Position is healthy");

        // Liquidador paga dívida, recebe colateral + bonus
        // (implementação similar ao Cap 10)
    }
}
```

### Price Feed Disponíveis

**Mainnet Ethereum** (exemplos):
```solidity
// ETH/USD: 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419
// BTC/USD: 0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c
// LINK/USD: 0x2c1d072e956AFFC0D435Cb7AC38EF18d24d9127c
// DAI/USD: 0xAed0c38402a5d19df6E4c03F4E2DceD6e29c1ee9

// Ver lista completa: https://docs.chain.link/data-feeds/price-feeds/addresses
```

💡 **Pro Tip**: Sempre verifique `updatedAt` timestamp para garantir que o preço não está stale!

```solidity
function getLatestPrice() public view returns (int) {
    (
        /* uint80 roundID */,
        int price,
        /* uint startedAt */,
        uint timeStamp,
        /* uint80 answeredInRound */
    ) = priceFeed.latestRoundData();

    // Verificar se preço foi atualizado recentemente (últimas 2 horas)
    require(block.timestamp - timeStamp < 2 hours, "Stale price");

    return price;
}
```

---

## 11.4 Chainlink VRF - Randomness

### O Problema da Randomness On-Chain

```solidity
// ❌ VULNERÁVEL - Minerador pode manipular!
function random() public view returns (uint256) {
    return uint256(keccak256(abi.encodePacked(block.timestamp, block.difficulty, msg.sender)));
}

// Minerador pode:
// 1. Calcular hash antes de minerar
// 2. Se resultado é ruim, não incluir o bloco
// 3. Tentar novamente
// → Não é verdadeiramente aleatório!
```

### Chainlink VRF (Verifiable Random Function)

**VRF = Randomness + Prova Criptográfica**

```
1. Smart contract pede número aleatório
2. Chainlink node gera número + prova criptográfica
3. Prova é verificada on-chain
4. Impossível manipular (matematicamente provável)
```

### Implementação: Loteria com VRF

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import "@chainlink/contracts/src/v0.8/interfaces/VRFCoordinatorV2Interface.sol";
import "@chainlink/contracts/src/v0.8/VRFConsumerBaseV2.sol";

contract Lottery is VRFConsumerBaseV2 {
    VRFCoordinatorV2Interface COORDINATOR;

    // VRF Config
    uint64 s_subscriptionId;
    bytes32 keyHash = 0x474e34a077df58807dbe9c96d3c009b23b3c6d0cce433e59bbf5b34f823bc56c;
    uint32 callbackGasLimit = 100000;
    uint16 requestConfirmations = 3;
    uint32 numWords = 1;

    // Lottery state
    address[] public players;
    address public recentWinner;
    uint256 public randomResult;

    mapping(uint256 => address) public requestIdToSender;

    event RequestedRandomness(uint256 indexed requestId);
    event WinnerPicked(address indexed winner, uint256 randomNumber);

    constructor(uint64 subscriptionId, address vrfCoordinator)
        VRFConsumerBaseV2(vrfCoordinator)
    {
        COORDINATOR = VRFCoordinatorV2Interface(vrfCoordinator);
        s_subscriptionId = subscriptionId;
    }

    /**
     * Entrar na loteria
     */
    function enter() public payable {
        require(msg.value >= 0.01 ether, "Minimum 0.01 ETH");
        players.push(msg.sender);
    }

    /**
     * Pedir número aleatório (apenas owner)
     */
    function pickWinner() public returns (uint256 requestId) {
        require(players.length > 0, "No players");

        // Request random number from Chainlink VRF
        requestId = COORDINATOR.requestRandomWords(
            keyHash,
            s_subscriptionId,
            requestConfirmations,
            callbackGasLimit,
            numWords
        );

        emit RequestedRandomness(requestId);
        return requestId;
    }

    /**
     * Callback do Chainlink VRF (chamado automaticamente)
     */
    function fulfillRandomWords(
        uint256 requestId,
        uint256[] memory randomWords
    ) internal override {
        // Pegar primeiro (e único) número aleatório
        randomResult = randomWords[0];

        // Escolher vencedor
        uint256 indexOfWinner = randomResult % players.length;
        address winner = players[indexOfWinner];
        recentWinner = winner;

        // Resetar loteria
        players = new address[](0);

        // Enviar prêmio
        (bool success, ) = winner.call{value: address(this).balance}("");
        require(success, "Transfer failed");

        emit WinnerPicked(winner, randomResult);
    }

    /**
     * View functions
     */
    function getNumberOfPlayers() public view returns (uint256) {
        return players.length;
    }

    function getPlayer(uint256 index) public view returns (address) {
        return players[index];
    }
}
```

### VRF Setup (Testnet)

**Passos**:
1. Ir para [vrf.chain.link](https://vrf.chain.link)
2. Criar subscription
3. Adicionar consumer (seu contrato)
4. Fundar subscription com LINK tokens
5. Deploy contrato com subscription ID

**Custos**: ~0.25 LINK por request (varia por network)

---

## 11.5 Chainlink Automation

### O Problema: Smart Contracts Não "Acordam"

```solidity
// ❌ Não funciona - ninguém chama essa função automaticamente
function dailyReward() public {
    require(block.timestamp >= lastRewardTime + 1 days);
    // Distribuir rewards...
}
```

**Em Web2**: `setInterval`, cron jobs, lambdas agendadas

**Em Web3**: Precisa de **keeper** (bot) para chamar função

### Chainlink Automation (Keepers)

**Como funciona**:
```
1. Você define condição (upkeepNeeded)
2. Chainlink nodes monitoram constantemente
3. Quando condição é true, nodes chamam performUpkeep
4. Você paga em LINK
```

### Implementação: Auto-Compound Staking

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import "@chainlink/contracts/src/v0.8/AutomationCompatible.sol";

contract AutoCompoundStaking is AutomationCompatibleInterface {
    // Staking state
    mapping(address => uint256) public stakedAmount;
    mapping(address => uint256) public rewards;
    uint256 public lastCompoundTime;
    uint256 public constant COMPOUND_INTERVAL = 1 days;

    event Compounded(uint256 totalRewards, uint256 timestamp);

    /**
     * Chainlink Automation chama para verificar se precisa executar
     */
    function checkUpkeep(bytes calldata /* checkData */)
        external
        view
        override
        returns (bool upkeepNeeded, bytes memory /* performData */)
    {
        upkeepNeeded = (block.timestamp - lastCompoundTime) > COMPOUND_INTERVAL;
        // performData pode ser usado para passar dados para performUpkeep
    }

    /**
     * Chainlink Automation executa quando checkUpkeep retorna true
     */
    function performUpkeep(bytes calldata /* performData */) external override {
        // Reverificar condição (segurança)
        require(
            (block.timestamp - lastCompoundTime) > COMPOUND_INTERVAL,
            "Too early"
        );

        // Calcular e distribuir rewards
        uint256 totalRewards = calculateRewards();
        distributeRewards(totalRewards);

        lastCompoundTime = block.timestamp;

        emit Compounded(totalRewards, block.timestamp);
    }

    function calculateRewards() internal view returns (uint256) {
        // Lógica de cálculo de rewards
        return address(this).balance / 100; // Exemplo: 1% do pool
    }

    function distributeRewards(uint256 totalRewards) internal {
        // Distribuir proporcionalmente para stakers
        // (implementação simplificada)
    }

    // Funções de staking...
    function stake() external payable {
        stakedAmount[msg.sender] += msg.value;
    }
}
```

### Casos de Uso Comuns

1. **Auto-compound**: Reinvestir rewards automaticamente
2. **Liquidações**: Chamar `liquidate()` quando health factor < threshold
3. **Vesting**: Liberar tokens em schedule
4. **Rebase tokens**: Ajustar supply periodicamente
5. **Oracle updates**: Atualizar preços de custom oracles
6. **Game mechanics**: Executar lógica de jogo temporizada

---

## 11.6 Custom Oracles

### Quando Construir Custom Oracle

✅ **Casos válidos**:
- Dados proprietários (seu próprio sistema)
- APIs não suportadas por Chainlink
- Proof-of-concept / testnet
- Dados que você controla e confia

❌ **NÃO use para**:
- Preços de mercado (use Chainlink)
- Randomness (use VRF)
- Produção com alto valor

### Implementação Simples: Push Oracle

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * Oracle simples: Trusted party atualiza preço off-chain
 * ⚠️ CENTRALIZADO - apenas para desenvolvimento/testes
 */
contract SimplePriceOracle {
    address public owner;
    uint256 public price;
    uint256 public lastUpdate;

    event PriceUpdated(uint256 newPrice, uint256 timestamp);

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    /**
     * Owner atualiza preço (off-chain bot chama isso)
     */
    function updatePrice(uint256 newPrice) external onlyOwner {
        price = newPrice;
        lastUpdate = block.timestamp;
        emit PriceUpdated(newPrice, block.timestamp);
    }

    /**
     * Contratos leem o preço
     */
    function getLatestPrice() external view returns (uint256) {
        require(block.timestamp - lastUpdate < 1 hours, "Stale price");
        return price;
    }
}
```

**Script off-chain** (Node.js):
```javascript
// update-oracle.js
const ethers = require('ethers');

async function updateOracle() {
  const provider = new ethers.JsonRpcProvider(RPC_URL);
  const wallet = new ethers.Wallet(PRIVATE_KEY, provider);
  const oracle = new ethers.Contract(ORACLE_ADDRESS, ABI, wallet);

  // Fetch price de API externa
  const response = await fetch('https://api.coinbase.com/v2/prices/ETH-USD/spot');
  const data = await response.json();
  const price = parseFloat(data.data.amount) * 1e8; // 8 decimals

  // Update on-chain
  const tx = await oracle.updatePrice(price);
  await tx.wait();

  console.log(`Price updated: $${price / 1e8}`);
}

// Rodar a cada 10 minutos
setInterval(updateOracle, 10 * 60 * 1000);
```

⚠️ **Warning**: Isso é **centralizado** - você é single point of failure!

### Oracle Descentralizado Simples

```solidity
/**
 * Múltiplos reporters, pega mediana
 * Mais seguro que single reporter
 */
contract DecentralizedOracle {
    struct Report {
        uint256 price;
        uint256 timestamp;
    }

    mapping(address => bool) public reporters;
    mapping(address => Report) public latestReports;
    address[] public reporterList;

    uint256 public constant MIN_REPORTERS = 3;

    event PriceReported(address indexed reporter, uint256 price);

    constructor(address[] memory _reporters) {
        for (uint i = 0; i < _reporters.length; i++) {
            reporters[_reporters[i]] = true;
            reporterList.push(_reporters[i]);
        }
    }

    modifier onlyReporter() {
        require(reporters[msg.sender], "Not a reporter");
        _;
    }

    /**
     * Cada reporter envia seu preço
     */
    function reportPrice(uint256 price) external onlyReporter {
        latestReports[msg.sender] = Report({
            price: price,
            timestamp: block.timestamp
        });

        emit PriceReported(msg.sender, price);
    }

    /**
     * Calcular mediana dos reports
     */
    function getMedianPrice() public view returns (uint256) {
        uint256[] memory prices = new uint256[](reporterList.length);
        uint256 validCount = 0;

        // Coletar prices válidos (< 1 hour old)
        for (uint i = 0; i < reporterList.length; i++) {
            Report memory report = latestReports[reporterList[i]];
            if (block.timestamp - report.timestamp < 1 hours) {
                prices[validCount] = report.price;
                validCount++;
            }
        }

        require(validCount >= MIN_REPORTERS, "Not enough recent reports");

        // Sort (bubble sort - ok para arrays pequenos)
        for (uint i = 0; i < validCount - 1; i++) {
            for (uint j = 0; j < validCount - i - 1; j++) {
                if (prices[j] > prices[j + 1]) {
                    uint256 temp = prices[j];
                    prices[j] = prices[j + 1];
                    prices[j + 1] = temp;
                }
            }
        }

        // Retornar mediana
        return prices[validCount / 2];
    }
}
```

---

## 11.7 Oracle Attacks

### 1. Price Manipulation (Flash Loan Attack)

**Cenário**:
```solidity
// ❌ VULNERÁVEL: Lending usa preço spot de DEX
contract VulnerableLending {
    function getCollateralValue() public view returns (uint256) {
        // Pega preço do próprio DEX
        (uint256 reserve0, uint256 reserve1) = pair.getReserves();
        uint256 price = (reserve1 * 1e18) / reserve0;
        return collateral * price;
    }
}

// Ataque:
// 1. Flash loan 10M USDC
// 2. Swap 10M USDC → ETH no DEX (preço sobe!)
// 3. Depositar 1 ETH como colateral (vale mais agora)
// 4. Emprestar muito USDC
// 5. Repagar flash loan
// 6. Profit!
```

**Proteção**:
```solidity
// ✅ SEGURO: Use TWAP ou Chainlink
function getCollateralValue() public view returns (uint256) {
    // Chainlink
    (, int price,,,) = priceFeed.latestRoundData();
    return collateral * uint256(price);

    // Ou TWAP
    return uniswapPair.price0CumulativeLast();
}
```

### 2. Stale Price

```solidity
// ❌ Não verifica idade do dado
function getPrice() public view returns (uint256) {
    (, int price,,,) = priceFeed.latestRoundData();
    return uint256(price);
}

// ✅ Verifica freshness
function getPrice() public view returns (uint256) {
    (
        uint80 roundID,
        int price,
        uint startedAt,
        uint timeStamp,
        uint80 answeredInRound
    ) = priceFeed.latestRoundData();

    require(timeStamp > 0, "Round not complete");
    require(answeredInRound >= roundID, "Stale answer");
    require(block.timestamp - timeStamp < 2 hours, "Price too old");

    return uint256(price);
}
```

### 3. Oracle Frontrunning

**Problema**: Attacker vê oracle update na mempool e frontrun.

```
1. Oracle submit novo preço ETH: $1900 → $2000
2. Attacker vê transação na mempool
3. Attacker paga mais gas para executar antes
4. Attacker: compra ETH a $1900, vende a $2000
```

**Proteção**:
- Commit-reveal scheme
- Flashbots / private mempools
- Time delays / TWAPs

### 4. Centralization Risk

**Problema**: Custom oracle com único reporter = single point of failure.

**Proteção**:
- Múltiplos reporters independentes
- Mediana em vez de média
- Stake/slashing para reporters desonestos
- **Melhor**: Use Chainlink (já descentralizado)

---

## 📖 Glossário

**Oracle**
> Serviço que fornece dados externos para smart contracts.
> **Analogia**: Como API gateway em Web2, mas com garantias criptográficas.
> **Por que existe**: Smart contracts não podem acessar internet.

**Oracle Problem**
> Dilema: blockchain é determinístico, mundo real não é.
> Como trazer dados externos sem quebrar consenso?

**Price Feed**
> Oracle especializado em fornecer preços de assets.
> **Exemplo**: ETH/USD, BTC/USD.

**VRF (Verifiable Random Function)**
> Randomness provável e verificável on-chain.
> **Por que precisa**: `block.timestamp` e similares são manipuláveis.

**TWAP (Time-Weighted Average Price)**
> Preço médio ponderado por tempo.
> **Por que usar**: Resistente a manipulação via flash loans.

**Aggregator**
> Contrato que combina dados de múltiplas fontes.
> **Exemplo**: Chainlink pega mediana de 7+ nodes.

**Keeper / Automation**
> Bot que chama funções de smart contract automaticamente.
> **Analogia**: Como cron job em servidor tradicional.

**Stale Price**
> Preço desatualizado (muito antigo).
> **Proteção**: Verificar `timeStamp` em price feeds.

**Flash Loan**
> Empréstimo sem colateral que deve ser pago na mesma transação.
> **Risco**: Pode ser usado para manipular oracles.

---

## 🔒 Security Checklist: Oracles

### Usando Price Oracles
- [ ] NUNCA use preço spot de único DEX
- [ ] Use Chainlink ou TWAP
- [ ] Verifique `updatedAt` timestamp (não aceitar stale)
- [ ] Verifique `answeredInRound >= roundID`
- [ ] Multiple oracles como fallback
- [ ] Circuit breakers para mudanças abruptas (>10% em curto período)

### Chainlink VRF
- [ ] Usar VRF v2 (mais recente)
- [ ] Callback gas limit adequado
- [ ] Subscription fundada com LINK
- [ ] Nunca confiar em `block.timestamp` ou `blockhash` para randomness

### Chainlink Automation
- [ ] `checkUpkeep` deve ser view/pure
- [ ] `performUpkeep` deve reverificar condições
- [ ] Gas limit adequado
- [ ] Subscription fundada

### Custom Oracles
- [ ] Múltiplos reporters independentes
- [ ] Mediana em vez de média (resistente a outliers)
- [ ] Verificar freshness dos dados
- [ ] Stake/incentivos para reporters honestos
- [ ] Considerar usar Chainlink em vez de custom

---

## 📝 Exercícios Práticos

### Exercício 1: DeFi com Chainlink Price Feeds

**Objetivo**: Integrar Chainlink em lending protocol.

**Descrição**:
Modificar contrato de lending do Cap 10 para:
1. Usar Chainlink Price Feed para ETH/USD e BTC/USD
2. Suportar múltiplos collaterals
3. Calcular health factor com preços reais
4. Implementar proteção contra stale prices

**Requisitos**:
- Verificar timestamp de update
- Fallback se oracle falhar
- Events para tracking de liquidações

<details>
<summary>💡 Dica</summary>

Mantenha mapeamento de collateral → price feed:
```solidity
mapping(address => AggregatorV3Interface) public priceFeeds;
```

Para cada collateral, consulte seu feed específico e some valores em USD.
</details>

---

### Exercício 2: NFT Lottery com VRF

**Objetivo**: Criar loteria on-chain com randomness verificável.

**Descrição**:
- Users entram pagando 0.01 ETH
- Mínimo 10 players
- Owner chama `pickWinner()` → Chainlink VRF escolhe vencedor
- Vencedor recebe 90% do pot, 10% para owner

**Desafio Extra**:
- Múltiplas loterias simultâneas
- Auto-start nova loteria após vencedor
- NFT como prêmio em vez de ETH

<details>
<summary>💡 Setup VRF</summary>

1. Deploy contrato
2. Criar subscription em vrf.chain.link
3. Adicionar contrato como consumer
4. Fund subscription com LINK
5. Call `pickWinner()` → esperar callback

Callback demora ~1 minuto (espera confirmações).
</details>

---

### Exercício 3: Auto-Compound Vault

**Objetivo**: Staking vault que reinveste rewards automaticamente.

**Descrição**:
- Users depositam tokens
- Acumula rewards
- Chainlink Automation chama `compound()` a cada 24h
- Rewards são reinvestidos no vault

**Tecnologias**:
- Chainlink Automation
- ERC4626 Vault pattern (opcional)
- Staking rewards calculation

<details>
<summary>💡 Estrutura</summary>

```solidity
function checkUpkeep() external view returns (bool) {
    return (block.timestamp - lastCompound) >= 1 days;
}

function performUpkeep() external {
    // 1. Claim rewards de protocolo externo
    // 2. Reinvestir no vault
    // 3. Atualizar shares dos users
}
```
</details>

---

## 📚 Recursos Adicionais

### Documentação Oficial

1. **[Chainlink Docs](https://docs.chain.link/)**
   - Referência completa de Data Feeds, VRF, Automation
   - Addresses de contratos em todas networks

2. **[Chainlink Price Feeds List](https://docs.chain.link/data-feeds/price-feeds/addresses)**
   - Todos feeds disponíveis (200+)
   - Mainnet, testnets, L2s

3. **[Chainlink VRF](https://docs.chain.link/vrf/v2/introduction)**
   - VRF v2 (mais eficiente que v1)
   - Subscription management

### Ferramentas

- **[vrf.chain.link](https://vrf.chain.link/)** - Manage VRF subscriptions
- **[automation.chain.link](https://automation.chain.link/)** - Manage Keepers
- **[Chainlink Faucets](https://faucets.chain.link/)** - LINK testnet

### Deep Dives

- **[Oracle Manipulation Attacks](https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles/)** - OpenZeppelin análise
- **[Chainlink 2.0 Whitepaper](https://chain.link/whitepaper)** - Arquitetura completa
- **[Rekt News: Oracle Edition](https://rekt.news/leaderboard/)** - Hacks via oracle manipulation

---

## 🎯 Próximos Passos

**Você agora entende**:
- ✅ Por que smart contracts precisam de oracles
- ✅ Como Chainlink funciona (descentralização + agregação)
- ✅ Price Feeds para DeFi
- ✅ VRF para randomness verificável
- ✅ Automation para executar tarefas periódicas
- ✅ Quando (e quando NÃO) construir custom oracle
- ✅ Principais ataques via oracle manipulation

**Próximo Capítulo**: [Capítulo 12 - Upgradeable Contracts](./EBOOK_CAPITULO_12_UPGRADEABLE.md)
- Proxy patterns (Transparent, UUPS, Diamond)
- Storage layout compatibility
- Governança on-chain (DAOs)
- Trade-offs: Upgradeability vs Immutability

**Aplicação Prática**:
Agora você pode construir DeFi apps completos:
- Cap 9 (Tokens) + Cap 10 (DeFi) + Cap 11 (Oracles) = Lending protocol production-ready
- Cap 11 (VRF) = Loteria, jogos, NFT minting aleatório
- Cap 11 (Automation) = Yield optimization, auto-liquidações

💡 **Projeto Sugerido**: DEX com Chainlink price feeds para exibir valor em USD de cada trade + auto-liquidation bot usando Automation.

---

**Autor**: Baseado em roadmap ITA Blockchain Club + Chainlink documentation
**Última Atualização**: 2025-11-14
