# Capítulo 4: Ambiente de Desenvolvimento Profissional

> **Para Desenvolvedores Experientes**: Se você já configurou projetos Node.js, Python ou Java, sabe que setup de ambiente importa. Em blockchain, isso é ainda mais crítico - um bug em produção pode custar milhões. Este capítulo cobre ferramentas que separam hobbyistas de profissionais.

---

## Índice
- [4.1 Remix - Quick Start (Mas Não Para Produção)](#41-remix---quick-start-mas-não-para-produção)
- [4.2 Hardhat vs Foundry - A Grande Decisão](#42-hardhat-vs-foundry---a-grande-decisão)
- [4.3 Estrutura de Projeto Profissional](#43-estrutura-de-projeto-profissional)
- [4.4 Testing Frameworks](#44-testing-frameworks)
- [4.5 Blockchain Local - Ganache e Anvil](#45-blockchain-local---ganache-e-anvil)
- [4.6 Deployment Scripts](#46-deployment-scripts)
- [4.7 Verificação de Contratos](#47-verificação-de-contratos)
- [4.8 CI/CD Básico](#48-cicd-básico)

---

## 4.1 Remix - Quick Start (Mas Não Para Produção)

### O que é Remix

**Remix IDE**: Editor online de Solidity da Ethereum Foundation.
- URL: https://remix.ethereum.org/
- 100% no navegador
- Zero setup necessário

### ✅ Quando Usar Remix

- Aprender Solidity (primeiros passos)
- Testar snippets rápidos
- Debug de contratos simples
- Workshops/demos

### ❌ Quando NÃO Usar Remix

- Projetos com múltiplos arquivos
- Testes automatizados complexos
- CI/CD
- Colaboração em equipe (sem Git integrado)
- Produção

### Quick Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Hello {
    string public message = "Hello, World!";

    function setMessage(string memory newMessage) public {
        message = newMessage;
    }
}
```

**Workflow no Remix**:
1. Criar arquivo `.sol`
2. Compilar (Ctrl+S ou botão "Compile")
3. Deploy (botão "Deploy" na aba "Deploy & Run")
4. Interagir (botão de funções aparece)

**Limitação**: Sem versionamento, sem testes robustos, sem integração externa.

---

## 4.2 Hardhat vs Foundry - A Grande Decisão

### Comparação Detalhada

| Critério | Hardhat | Foundry |
|----------|---------|---------|
| **Linguagem** | JavaScript/TypeScript | Solidity |
| **Performance** | ~10s para compilar + testar | ~100ms para compilar + testar |
| **Testes** | Mocha/Chai (JavaScript) | Solidity (nativo) |
| **Plugins** | 100+ plugins | Poucos (ainda crescendo) |
| **Curva de Aprendizado** | Baixa (se já sabe JS) | Média (tudo em Solidity) |
| **Debugging** | `console.log` em Solidity | `forge debug` (debugger interativo) |
| **Fuzzing** | Não nativo | Nativo (`forge fuzz`) |
| **Fork Testing** | Sim (via plugin) | Sim (nativo, muito rápido) |
| **Comunidade** | Maior (mais antigo) | Crescente rapidamente |
| **Deploy Scripts** | JavaScript | Solidity (via script contracts) |
| **Gas Reports** | Via plugin | Nativo |
| **Popularidade** | ~80% dos projetos | ~20% (mas crescendo) |

### Quando Escolher Hardhat

✅ **Use Hardhat se**:
- Time já forte em JavaScript/TypeScript
- Precisa de plugins específicos (muitos só existem para Hardhat)
- Projeto legado já usa Hardhat
- Quer console.log no Solidity (muito útil para debug)

**Exemplo de projetos que usam**:
- Uniswap V3
- Aave
- Compound

### Quando Escolher Foundry

✅ **Use Foundry se**:
- Prioriza velocidade (testes 100x mais rápidos)
- Prefere tudo em Solidity (menos context switching)
- Quer fuzzing robusto
- Está começando projeto novo
- Fork testing é crítico

**Exemplo de projetos que usam**:
- Uniswap V4 (migraram de Hardhat!)
- Optimism
- zkSync

### Minha Recomendação

**Para este ebook**: Vou focar em **Foundry** por:
1. Velocidade (feedback loops rápidos)
2. Tudo em Solidity (menos linguagens para aprender)
3. Tendência da indústria (novos projetos estão migrando)

**Mas**: Vou mencionar Hardhat quando relevante. Conhecer ambos é valioso.

---

## 4.3 Estrutura de Projeto Profissional

### Setup Foundry

**Instalação**:
```bash
# macOS/Linux
curl -L https://foundry.paradigm.xyz | bash
foundryup

# Verificar instalação
forge --version
```

**Criar projeto**:
```bash
forge init my-project
cd my-project
```

**Estrutura gerada**:
```
my-project/
├── foundry.toml          # Configuração do Foundry
├── lib/                  # Dependencies (via git submodules)
│   └── forge-std/        # Standard library de testes
├── script/               # Deploy scripts
│   └── Counter.s.sol     # Exemplo de script
├── src/                  # Smart contracts
│   └── Counter.sol       # Exemplo de contrato
└── test/                 # Testes
    └── Counter.t.sol     # Exemplo de teste
```

### foundry.toml - Configuração

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc_version = "0.8.19"
optimizer = true
optimizer_runs = 200

# EVM version (default = london)
evm_version = "paris"

# Verbosity durante testes
verbosity = 2

# Gas reports
gas_reports = ["*"]

[profile.ci]
fuzz = { runs = 10000 }  # Mais runs em CI

[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"

[etherscan]
mainnet = { key = "${ETHERSCAN_API_KEY}" }
```

### Adicionar Dependencies

Foundry usa **git submodules**:

```bash
# Adicionar OpenZeppelin Contracts
forge install OpenZeppelin/openzeppelin-contracts

# Adicionar Solmate (gas-optimized contracts)
forge install transmissions11/solmate

# Ver dependencies
cat .gitmodules
```

**Usar em código**:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import "solmate/tokens/ERC20.sol"; // Alternativa mais otimizada

contract MyToken is ERC20 {
    constructor() ERC20("MyToken", "MTK") {
        _mint(msg.sender, 1000000 * 10**18);
    }
}
```

---

## 4.4 Testing Frameworks

### Testes em Foundry (Solidity)

**Estrutura básica**:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;

    // Setup (executado antes de cada teste)
    function setUp() public {
        counter = new Counter();
    }

    // Teste simples
    function testIncrement() public {
        counter.increment();
        assertEq(counter.count(), 1);
    }

    // Teste com parâmetros (fuzzing)
    function testSetCount(uint256 x) public {
        counter.setCount(x);
        assertEq(counter.count(), x);
    }

    // Teste de revert
    function testCannotDecrementBelowZero() public {
        vm.expectRevert();
        counter.decrement(); // count = 0, não pode decrementar
    }
}
```

**Rodar testes**:
```bash
# Todos os testes
forge test

# Testes específicos
forge test --match-contract CounterTest
forge test --match-test testIncrement

# Com verbosidade
forge test -vvvv  # Muito verboso (mostra cada opcode!)

# Com gas report
forge test --gas-report
```

### Cheatcodes (vm.*)

**Foundry tem "cheatcodes" poderosos**:

```solidity
contract CheatcodeExample is Test {
    function testPrank() public {
        address alice = address(0x1);

        // vm.prank: próxima chamada vem de 'alice'
        vm.prank(alice);
        someContract.doSomething(); // msg.sender = alice

        // vm.startPrank: todas chamadas vêm de 'alice' até stopPrank
        vm.startPrank(alice);
        someContract.doSomething();
        someContract.doMore();
        vm.stopPrank();
    }

    function testWarp() public {
        // vm.warp: mudar block.timestamp
        vm.warp(1000);
        assertEq(block.timestamp, 1000);

        // Avançar 1 dia
        vm.warp(block.timestamp + 1 days);
    }

    function testDeal() public {
        address user = address(0x1);

        // vm.deal: dar ETH para address
        vm.deal(user, 100 ether);
        assertEq(user.balance, 100 ether);
    }

    function testMockCall() public {
        // vm.mockCall: mockar chamada externa
        address token = address(0x123);
        vm.mockCall(
            token,
            abi.encodeWithSelector(IERC20.balanceOf.selector, alice),
            abi.encode(1000)
        );

        // Agora token.balanceOf(alice) retorna 1000
    }

    function testExpectEmit() public {
        // Verificar que evento foi emitido
        vm.expectEmit(true, true, false, true);
        emit Transfer(alice, bob, 100);

        token.transfer(bob, 100); // Deve emitir Transfer(alice, bob, 100)
    }
}
```

**Mais cheatcodes úteis**:
- `vm.roll(blockNumber)`: mudar block.number
- `vm.fee(gasPrice)`: mudar tx.gasprice
- `vm.expectRevert(bytes)`: esperar revert específico
- `vm.assume(bool)`: assumir condição (para fuzzing)
- `vm.label(address, string)`: dar nome legível para address

### Assertions Disponíveis

```solidity
// Igualdade
assertEq(a, b);
assertEq(a, b, "Custom error message");

// Booleanos
assertTrue(condition);
assertFalse(condition);

// Comparações
assertGt(a, b);  // a > b
assertGe(a, b);  // a >= b
assertLt(a, b);  // a < b
assertLe(a, b);  // a <= b

// Diferença aproximada (útil para cálculos)
assertApproxEqAbs(a, b, maxDelta);     // |a - b| <= maxDelta
assertApproxEqRel(a, b, maxPercentDelta); // |a - b| / b <= maxPercentDelta
```

---

## 4.5 Blockchain Local - Ganache e Anvil

### Anvil (Foundry)

**Anvil**: Blockchain local do Foundry (substituto do Ganache).

**Iniciar**:
```bash
# Inicia blockchain local na porta 8545
anvil

# Com fork da mainnet
anvil --fork-url $MAINNET_RPC_URL

# Com block time fixo (default = instant)
anvil --block-time 12  # 12 segundos como Ethereum

# Com contas específicas
anvil --accounts 20 --balance 10000
```

**Características**:
- ✅ Instant mining (transações instantâneas)
- ✅ Fork de mainnet em segundos
- ✅ 10 contas pré-financiadas
- ✅ Determinístico (mesmas private keys sempre)

**Contas default** (NUNCA use em mainnet!):
```
Account #0: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
Private Key: 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

Account #1: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
Private Key: 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d

...
```

### Ganache (Hardhat/Truffle)

**Se estiver usando Hardhat**:
```bash
# Hardhat tem node local built-in
npx hardhat node

# Automático em testes
npx hardhat test  # Inicia node temporário
```

---

## 4.6 Deployment Scripts

### Deploy Script em Foundry

```solidity
// script/DeployToken.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "../src/MyToken.sol";

contract DeployToken is Script {
    function run() external {
        // Pegar private key de env variable
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");

        // Começar broadcast (transações reais)
        vm.startBroadcast(deployerPrivateKey);

        // Deploy contrato
        MyToken token = new MyToken("My Token", "MTK");

        console.log("Token deployed at:", address(token));

        vm.stopBroadcast();
    }
}
```

**Executar**:
```bash
# Simular (dry-run)
forge script script/DeployToken.s.sol

# Deploy na testnet
forge script script/DeployToken.s.sol \
    --rpc-url $SEPOLIA_RPC_URL \
    --broadcast \
    --verify \  # Verificar no Etherscan
    -vvvv

# Deploy com mais opções
forge script script/DeployToken.s.sol \
    --rpc-url $SEPOLIA_RPC_URL \
    --broadcast \
    --verify \
    --etherscan-api-key $ETHERSCAN_API_KEY \
    --gas-estimate-multiplier 120  # 20% buffer no gas
```

**Environment variables** (.env):
```bash
# .env
PRIVATE_KEY=0x... # NUNCA commitar isso no Git!
SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/YOUR_KEY
MAINNET_RPC_URL=https://mainnet.infura.io/v3/YOUR_KEY
ETHERSCAN_API_KEY=YOUR_ETHERSCAN_KEY
```

**Carregar env**:
```bash
source .env
# ou
forge script ... --env-file .env
```

### Deploy Multi-Step

```solidity
contract DeployFull is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");

        vm.startBroadcast(deployerPrivateKey);

        // 1. Deploy token
        MyToken token = new MyToken();

        // 2. Deploy exchange com endereço do token
        Exchange exchange = new Exchange(address(token));

        // 3. Configurar permissões
        token.grantRole(token.MINTER_ROLE(), address(exchange));

        // 4. Transferir ownership
        token.transferOwnership(vm.envAddress("OWNER_ADDRESS"));

        vm.stopBroadcast();

        // Log addresses (para salvar)
        console.log("Token:", address(token));
        console.log("Exchange:", address(exchange));
    }
}
```

---

## 4.7 Verificação de Contratos

### Por Que Verificar

**Contrato não-verificado**: Apenas bytecode visível
**Contrato verificado**: Código-fonte e ABI visíveis no Etherscan

**Benefícios**:
- ✅ Usuários podem auditar código
- ✅ Interagir diretamente no Etherscan
- ✅ Transparência e confiança
- ✅ Debugging mais fácil

### Verificar com Foundry

**Método 1: Durante deploy**:
```bash
forge script script/Deploy.s.sol \
    --rpc-url $SEPOLIA_RPC_URL \
    --broadcast \
    --verify  # ← Verifica automaticamente
```

**Método 2: Depois do deploy**:
```bash
forge verify-contract \
    --chain-id 11155111 \  # Sepolia
    --etherscan-api-key $ETHERSCAN_API_KEY \
    0x... \  # Contract address
    src/MyToken.sol:MyToken  # Path:ContractName
```

**Com constructor arguments**:
```bash
forge verify-contract \
    0x... \
    src/MyToken.sol:MyToken \
    --constructor-args $(cast abi-encode "constructor(string,string)" "MyToken" "MTK")
```

### Verificar Proxy (Complexo)

Para contratos upgradeable (proxy pattern):
```bash
# 1. Verificar implementation
forge verify-contract 0xIMPL... Implementation.sol:Implementation

# 2. Verificar proxy (manualmente no Etherscan)
# Etherscan → More Options → "Is this a proxy?" → Verify
```

---

## 4.8 CI/CD Básico

### GitHub Actions para Foundry

**.github/workflows/test.yml**:
```yaml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive  # Foundry usa submodules

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Run tests
        run: forge test -vvv

      - name: Check gas snapshots
        run: forge snapshot --check

  coverage:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Run coverage
        run: forge coverage --report lcov

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./lcov.info
```

### Gas Snapshots

**Track gas usage over time**:
```bash
# Criar snapshot
forge snapshot

# Compara com snapshot anterior
forge snapshot --diff

# Falhar se gas aumentou
forge snapshot --check
```

Gera arquivo `.gas-snapshot`:
```
testTransfer() (gas: 51234)
testMint() (gas: 73456)
testBurn() (gas: 42000)
```

**Em CI**: Falha se PR aumenta gas consumption.

---

## 📖 Glossário Consolidado

**Forge**
> Ferramenta de CLI do Foundry para compilar, testar e fazer deploy.
>
> Comandos principais:
> - `forge build`: compilar
> - `forge test`: rodar testes
> - `forge create`: deploy simples
> - `forge script`: deploy complexo

**Cast**
> Ferramenta de CLI do Foundry para interagir com blockchain.
>
> Exemplos:
> - `cast call`: fazer call (read-only)
> - `cast send`: fazer transação
> - `cast balance`: ver balance
> - `cast abi-encode`: encodar dados

**Anvil**
> Blockchain local do Foundry (substituto do Ganache).

**Cheatcodes**
> Funções especiais (`vm.*`) disponíveis em testes Foundry que permitem manipular blockchain state.

**Fuzzing**
> Testes com inputs aleatórios para encontrar edge cases.
>
> Foundry gera automaticamente milhares de inputs para funções de teste.

**Fork Testing**
> Testar contra state real da mainnet/testnet.
>
> Permite interagir com contratos existentes (Uniswap, Aave, etc.) em testes.

---

## 🔒 Security Checklist: Ambiente de Dev

- [ ] **Private keys**: NUNCA no código ou Git (.gitignore .env!)
- [ ] **RPC URLs**: Usar Infura/Alchemy, não expor publicamente
- [ ] **Gas limits**: Set apropriadamente para prevenir stuck transactions
- [ ] **Testnet first**: Sempre testar em testnet antes de mainnet
- [ ] **Verified contracts**: Verificar no Etherscan após deploy
- [ ] **Multi-sig**: Considerar para contracts com valor alto
- [ ] **Timelocks**: Para funções críticas

---

## 📝 Exercícios Práticos

### Exercício 1: Setup e Primeiro Test

1. Instalar Foundry
2. Criar projeto
3. Escrever contrato ERC20 básico
4. Escrever 5 testes
5. Rodar com `forge test -vvv`

<details>
<summary>✅ Solução</summary>

```bash
# 1. Install
curl -L https://foundry.paradigm.xyz | bash
foundryup

# 2. Create
forge init my-token
cd my-token

# 3. Contract (src/MyToken.sol)
# (ver exemplo de ERC20 no Cap 9)

# 4. Tests (test/MyToken.t.sol)
# (ver exemplo acima)

# 5. Run
forge test -vvv
```
</details>

### Exercício 2: Fork Test com Uniswap

Escrever teste que:
1. Faz fork da mainnet
2. Swapa ETH por USDC no Uniswap
3. Verifica que recebeu USDC

<details>
<summary>💡 Dica</summary>

- Use `vm.createSelectFork()`
- Interface ISwapRouter do Uniswap V3
- Pode precisar de ~1 ETH de test balance (`vm.deal`)
</details>

---

## 📚 Recursos Adicionais

### Documentação
1. **[Foundry Book](https://book.getfoundry.sh/)** - Documentação oficial completa
2. **[Foundry GitHub](https://github.com/foundry-rs/foundry)** - Código-fonte e exemplos
3. **[Hardhat Docs](https://hardhat.org/docs)** - Se escolher Hardhat

### Ferramentas
- **[Tenderly](https://tenderly.co/)** - Debugger visual para transações
- **[OpenZeppelin Defender](https://defender.openzeppelin.com/)** - Deploy seguro e monitoring

---

## 🎯 Próximos Passos

**💡 Foundry Atingiu Maturidade (2025)**:

Foundry v1.0 foi lançado em **fevereiro de 2025**, marcando estabilidade de API e produção-ready após anos de desenvolvimento. Desde então:

- **Versão atual**: v1.4.4 (Outubro 2025) - bugfixes e melhorias contínuas
- **Adoção**: Projetos grandes migrando (Uniswap V4, Optimism, zkSync)
- **Estabilidade**: API estável, sem breaking changes significativos desde v1.0
- **Nightly builds**: Disponíveis para features experimentais

**Recomendação**: Foundry está maduro para produção. Se você está começando um novo projeto em Nov/2025, Foundry é uma escolha sólida e moderna.

---

Com ambiente configurado, você está pronto para:

→ **Capítulo 5**: Design Patterns em Solidity
- Checks-Effects-Interactions
- Factory, Proxy, etc.

→ **Capítulo 6**: Testing Avançado
- Unit, Integration, Fork
- Fuzzing profissional

---

**Autor**: Baseado em experiência com Foundry e Hardhat em produção
**Última Atualização**: 2025-11-17 (Revisão técnica: Foundry v1.4.4, comandos validados, contexto v1.0 milestone)
