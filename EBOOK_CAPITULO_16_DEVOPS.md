# Capítulo 16: DevOps para Smart Contracts

## Introdução: CI/CD na Era Blockchain

Se você vem do desenvolvimento Web2, está acostumado com pipelines de CI/CD que:
- Rodam testes automaticamente em cada PR
- Fazem deploy automático para staging/production
- Geram relatórios de coverage
- Validam code quality com linters

**No desenvolvimento blockchain, DevOps tem desafios únicos:**

| Aspecto | Web2 | Web3 (Blockchain) |
|---------|------|-------------------|
| **Deploy** | Substituir código no servidor | Immutable - novo contrato ou upgrade via proxy |
| **Rollback** | Git revert + redeploy | Impossível (código é imutável) - precisa de upgrade |
| **Testing** | Unit + Integration | Unit + Integration + **Fork testing** + Fuzzing |
| **Cost reporting** | Infraestrutura (AWS, etc.) | **Gas cost** em cada função |
| **Security** | OWASP, dependency scan | OWASP + **Slither/Mythril** + audit |
| **Secrets** | API keys, DB passwords | **Private keys** (nunca em CI) |
| **Environment** | Dev/Staging/Prod servers | **Testnets** (Goerli, Sepolia) → Mainnet |
| **Monitoring** | APM, logs | **Tenderly, Defender** (eventos on-chain) |

**Por que DevOps é crítico em blockchain:**
1. **Immutability**: Erro em produção = dinheiro perdido (não há rollback)
2. **Gas costs**: Pipeline precisa reportar custos para evitar surpresas
3. **Security**: Vulnerabilidades podem drenar fundos - precisa de análise estática
4. **Multi-network**: Mesmo código, múltiplas chains (mainnet, L2s, testnets)

**Neste capítulo, você vai aprender:**
- ✅ Configurar CI/CD com GitHub Actions para smart contracts
- ✅ Rodar testes automaticamente (unit, integration, fork, fuzz)
- ✅ Gerar relatórios de gas e coverage em PRs
- ✅ Integrar Slither/Mythril para security scanning
- ✅ Automatizar deployment para testnets
- ✅ Gerenciar secrets e private keys com segurança
- ✅ Criar workflows multi-ambiente (dev → staging → prod)
- ✅ Integrar com monitoring (Tenderly)

---

## Índice

1. [CI/CD Pipeline Básico](#1-cicd-pipeline-básico)
2. [Testing Automation](#2-testing-automation)
3. [Gas Reporting](#3-gas-reporting)
4. [Security Scanning](#4-security-scanning)
5. [Coverage e Code Quality](#5-coverage-e-code-quality)
6. [Deployment Automation](#6-deployment-automation)
7. [Secrets Management](#7-secrets-management)
8. [Multi-Environment Strategy](#8-multi-environment-strategy)
9. [Contract Verification](#9-contract-verification)
10. [Monitoring Integration](#10-monitoring-integration)
11. [Best Practices](#11-best-practices)
12. [Glossário](#12-glossário)
13. [Exercícios](#13-exercícios)
14. [Recursos](#14-recursos)
15. [Próximos Passos](#15-próximos-passos)

---

## 1. CI/CD Pipeline Básico

### Workflow GitHub Actions - Foundry

Crie `.github/workflows/test.yml`:

```yaml
name: Test Smart Contracts

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    name: Foundry Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          submodules: recursive  # Para lib/ dependencies

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
        with:
          version: nightly

      - name: Run tests
        run: forge test -vvv
        env:
          # Fork testing precisa de RPC
          MAINNET_RPC_URL: ${{ secrets.MAINNET_RPC_URL }}
          SEPOLIA_RPC_URL: ${{ secrets.SEPOLIA_RPC_URL }}

      - name: Check contract sizes
        run: forge build --sizes
```

**O que está acontecendo:**
1. **Trigger**: Roda em push para `main`/`develop` e em todos os PRs
2. **Foundry install**: Usa action oficial (`foundry-toolchain`)
3. **Tests**: `forge test -vvv` (verbose para ver falhas)
4. **Contract sizes**: Verifica se algum contrato > 24KB (limite da EVM)

### Workflow para Hardhat (alternativa)

```yaml
name: Hardhat Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Compile contracts
        run: npx hardhat compile

      - name: Run tests
        run: npx hardhat test
        env:
          MAINNET_RPC_URL: ${{ secrets.MAINNET_RPC_URL }}

      - name: Check contract sizes
        run: npx hardhat size-contracts
```

---

## 2. Testing Automation

### Workflow Completo com Múltiplos Tipos de Teste

```yaml
name: Comprehensive Tests

on: [push, pull_request]

jobs:
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Run unit tests
        run: forge test --match-path "test/unit/**/*.sol" -vvv

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Run integration tests
        run: forge test --match-path "test/integration/**/*.sol" -vvv
        env:
          MAINNET_RPC_URL: ${{ secrets.MAINNET_RPC_URL }}

  fork-tests:
    name: Fork Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Run fork tests (Mainnet)
        run: |
          forge test \
            --match-path "test/fork/**/*.sol" \
            --fork-url ${{ secrets.MAINNET_RPC_URL }} \
            --fork-block-number 18000000 \
            -vvv

  fuzz-tests:
    name: Fuzz Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Run fuzz tests
        run: |
          forge test \
            --match-path "test/fuzz/**/*.sol" \
            --fuzz-runs 10000 \
            -vvv
```

**Estratégia:**
- **Paralelização**: Cada tipo de teste em job separado (roda em paralelo)
- **Fork tests**: Fixa block number para resultados consistentes
- **Fuzz runs**: 10k runs (ajuste para CI - no local pode usar 50k+)

### Fail Fast vs Run All

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      # fail-fast: false => roda todos os testes mesmo se um falhar
      # Útil para ver todos os problemas de uma vez
      fail-fast: false
      matrix:
        test-type: [unit, integration, fork, fuzz]

    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Run ${{ matrix.test-type }} tests
        run: |
          forge test \
            --match-path "test/${{ matrix.test-type }}/**/*.sol" \
            -vvv
```

---

## 3. Gas Reporting

### Gas Report Automático em PRs

**1. Foundry gas reporting:**

```yaml
name: Gas Report

on:
  pull_request:
    branches: [main]

jobs:
  gas-report:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write  # Permite comentar no PR

    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Run tests with gas report
        run: forge test --gas-report > gas-report.txt

      - name: Comment PR with gas report
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const gasReport = fs.readFileSync('gas-report.txt', 'utf8');

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## ⛽ Gas Report\n\`\`\`\n${gasReport}\n\`\`\``
            });
```

**Output esperado no PR:**

```
⛽ Gas Report

╭────────────────────────┬─────────────────┬────────┬────────┬────────┬─────────╮
│ Contract               │ Function        │ Min    │ Avg    │ Max    │ Calls   │
├────────────────────────┼─────────────────┼────────┼────────┼────────┼─────────┤
│ MyToken                │ transfer        │ 51234  │ 51234  │ 51234  │ 42      │
│ MyToken                │ approve         │ 46123  │ 46123  │ 46123  │ 15      │
│ MyStaking              │ stake           │ 78456  │ 82345  │ 89123  │ 28      │
│ MyStaking              │ unstake         │ 34567  │ 45678  │ 56789  │ 18      │
╰────────────────────────┴─────────────────┴────────┴────────┴────────┴─────────╯
```

### Gas Diff (comparar com base branch)

```yaml
- name: Checkout base branch
  uses: actions/checkout@v4
  with:
    ref: ${{ github.base_ref }}
    path: base

- name: Run tests on base
  working-directory: base
  run: forge test --gas-report > ../gas-base.txt

- name: Checkout PR branch
  uses: actions/checkout@v4
  with:
    path: pr

- name: Run tests on PR
  working-directory: pr
  run: forge test --gas-report > ../gas-pr.txt

- name: Compare gas reports
  run: |
    echo "## Gas Diff" > gas-diff.md
    # Script para comparar gas-base.txt com gas-pr.txt
    # Exibir diferenças (functions que aumentaram/diminuíram gas)
```

### Hardhat Gas Reporter

Para projetos Hardhat, use `hardhat-gas-reporter`:

```javascript
// hardhat.config.js
require("hardhat-gas-reporter");

module.exports = {
  gasReporter: {
    enabled: process.env.REPORT_GAS === "true",
    currency: "USD",
    coinmarketcap: process.env.COINMARKETCAP_API_KEY,
    outputFile: "gas-report.txt",
    noColors: true
  }
};
```

```yaml
- name: Run tests with gas report
  run: REPORT_GAS=true npx hardhat test
  env:
    COINMARKETCAP_API_KEY: ${{ secrets.COINMARKETCAP_API_KEY }}
```

---

## 4. Security Scanning

### Slither Integration

```yaml
name: Security Scan

on: [push, pull_request]

jobs:
  slither:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install Slither
        run: pip3 install slither-analyzer

      - name: Install Foundry (para compilar)
        uses: foundry-rs/foundry-toolchain@v1

      - name: Run Slither
        run: |
          slither . \
            --exclude-dependencies \
            --exclude-informational \
            --exclude-low \
            --json slither-report.json
        continue-on-error: true  # Não falhar build, mas gerar report

      - name: Upload Slither report
        uses: actions/upload-artifact@v3
        with:
          name: slither-report
          path: slither-report.json

      - name: Comment PR with findings
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = JSON.parse(fs.readFileSync('slither-report.json'));

            const highSeverity = report.results.detectors.filter(d => d.impact === 'High');

            if (highSeverity.length > 0) {
              const body = `## 🚨 Slither encontrou ${highSeverity.length} issue(s) de alta severidade:\n\n` +
                highSeverity.map(issue => `- **${issue.check}**: ${issue.description}`).join('\n');

              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body
              });
            }
```

### Mythril Integration

```yaml
jobs:
  mythril:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install Mythril
        run: pip3 install mythril

      - name: Run Mythril on critical contracts
        run: |
          myth analyze src/MyVault.sol \
            --solv 0.8.20 \
            --execution-timeout 300 \
            > mythril-report.txt
        continue-on-error: true

      - name: Upload Mythril report
        uses: actions/upload-artifact@v3
        with:
          name: mythril-report
          path: mythril-report.txt
```

**Quando rodar security scanning:**
- ✅ Em **todo PR** (Slither rápido)
- ✅ Antes de **deploy para testnet** (Mythril + Slither)
- ✅ Antes de **audit profissional** (todos os scanners)
- ❌ Não rodar Mythril em CI para contratos grandes (muito lento)

---

## 5. Coverage e Code Quality

### Coverage Report com Foundry

```yaml
jobs:
  coverage:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Run coverage
        run: forge coverage --report lcov

      - name: Upload to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./lcov.info
          flags: foundry
          fail_ci_if_error: true
          token: ${{ secrets.CODECOV_TOKEN }}
```

**Configurar Codecov:**
1. Criar conta em [codecov.io](https://codecov.io)
2. Adicionar repo e obter token
3. Adicionar `CODECOV_TOKEN` nos secrets do GitHub
4. Badge no README:
   ```markdown
   [![codecov](https://codecov.io/gh/username/repo/branch/main/graph/badge.svg)](https://codecov.io/gh/username/repo)
   ```

### Solhint (Linting)

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install Solhint
        run: npm install -g solhint

      - name: Run Solhint
        run: solhint 'src/**/*.sol'
```

**`.solhint.json` recomendado:**

```json
{
  "extends": "solhint:recommended",
  "rules": {
    "compiler-version": ["error", "^0.8.0"],
    "func-visibility": ["warn", {"ignoreConstructors": true}],
    "not-rely-on-time": "off",
    "no-empty-blocks": "error",
    "no-unused-vars": "error",
    "avoid-low-level-calls": "warn",
    "avoid-sha3": "warn",
    "avoid-suicide": "error",
    "avoid-throw": "error"
  }
}
```

---

## 6. Deployment Automation

### Deploy para Testnet (Sepolia)

```yaml
name: Deploy to Sepolia

on:
  push:
    branches: [develop]  # Deploy develop → Sepolia
  workflow_dispatch:  # Permite deploy manual

jobs:
  deploy-testnet:
    runs-on: ubuntu-latest
    environment: sepolia  # GitHub Environment (para secrets)

    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Deploy contracts
        run: |
          forge script script/Deploy.s.sol \
            --rpc-url ${{ secrets.SEPOLIA_RPC_URL }} \
            --private-key ${{ secrets.DEPLOYER_PRIVATE_KEY }} \
            --broadcast \
            --verify \
            --etherscan-api-key ${{ secrets.ETHERSCAN_API_KEY }}

      - name: Save deployment addresses
        run: |
          # Broadcast cria broadcast/Deploy.s.sol/11155111/run-latest.json
          # (11155111 é chain ID da Sepolia)
          cat broadcast/Deploy.s.sol/11155111/run-latest.json \
            | jq '.transactions[] | {contract: .contractName, address: .contractAddress}' \
            > deployment-sepolia.json

      - name: Upload deployment info
        uses: actions/upload-artifact@v3
        with:
          name: deployment-sepolia
          path: deployment-sepolia.json

      - name: Comment PR with deployment
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const deployment = fs.readFileSync('deployment-sepolia.json', 'utf8');

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 🚀 Deployed to Sepolia\n\`\`\`json\n${deployment}\n\`\`\``
            });
```

### Deploy para Mainnet (Manual Approval)

```yaml
name: Deploy to Mainnet

on:
  workflow_dispatch:  # Apenas manual
    inputs:
      confirmation:
        description: 'Type "DEPLOY_TO_MAINNET" to confirm'
        required: true

jobs:
  deploy-mainnet:
    runs-on: ubuntu-latest
    environment: mainnet  # Requer approval no GitHub

    steps:
      - name: Validate confirmation
        run: |
          if [ "${{ github.event.inputs.confirmation }}" != "DEPLOY_TO_MAINNET" ]; then
            echo "❌ Confirmation failed. You must type DEPLOY_TO_MAINNET"
            exit 1
          fi

      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Run full test suite
        run: forge test
        env:
          MAINNET_RPC_URL: ${{ secrets.MAINNET_RPC_URL }}

      - name: Run Slither
        run: |
          pip3 install slither-analyzer
          slither . --exclude-dependencies

      - name: Deploy to Mainnet
        run: |
          forge script script/Deploy.s.sol \
            --rpc-url ${{ secrets.MAINNET_RPC_URL }} \
            --private-key ${{ secrets.MAINNET_DEPLOYER_KEY }} \
            --broadcast \
            --verify \
            --etherscan-api-key ${{ secrets.ETHERSCAN_API_KEY }} \
            --slow  # Espera confirmações

      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: '🚀 Mainnet deployment completed!'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

**GitHub Environment setup:**
1. Settings → Environments → New environment (`mainnet`)
2. Add protection rules:
   - ✅ Required reviewers (pelo menos 2 pessoas)
   - ✅ Wait timer (15 minutos para rever)
3. Add environment secrets (`MAINNET_RPC_URL`, `MAINNET_DEPLOYER_KEY`)

---

## 7. Secrets Management

### ❌ NUNCA faça isso:

```yaml
# ERRADO - private key hardcoded
- name: Deploy
  run: |
    forge script Deploy.s.sol \
      --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

### ✅ Jeito correto:

**1. GitHub Secrets:**

- Repository → Settings → Secrets → Actions
- Add secret: `DEPLOYER_PRIVATE_KEY`

```yaml
- name: Deploy
  run: |
    forge script Deploy.s.sol \
      --private-key ${{ secrets.DEPLOYER_PRIVATE_KEY }}
```

**2. Environment-specific secrets:**

```yaml
jobs:
  deploy:
    environment: ${{ github.ref == 'refs/heads/main' && 'mainnet' || 'sepolia' }}
    steps:
      - name: Deploy
        run: |
          forge script Deploy.s.sol \
            --rpc-url ${{ secrets.RPC_URL }} \
            --private-key ${{ secrets.DEPLOYER_KEY }}
```

**3. Usando `.env` localmente (NUNCA commitar):**

```bash
# .env (adicionar ao .gitignore)
DEPLOYER_PRIVATE_KEY=0x...
MAINNET_RPC_URL=https://...
ETHERSCAN_API_KEY=...
```

```solidity
// script/Deploy.s.sol
contract DeployScript is Script {
    function run() external {
        // Lê do .env (local) ou environment variable (CI)
        uint256 deployerPrivateKey = vm.envUint("DEPLOYER_PRIVATE_KEY");

        vm.startBroadcast(deployerPrivateKey);
        // Deploy logic...
        vm.stopBroadcast();
    }
}
```

**`.gitignore` obrigatório:**

```
# .gitignore
.env
broadcast/*/31337/  # Localhost
broadcast/*/*.json  # Deployment artifacts sensíveis
cache/
out/
node_modules/
```

### Hardware Wallet no CI (Enterprise)

Para produção, considere:
- **AWS KMS**: Chave privada gerenciada pela AWS
- **Google Cloud KMS**: Similar ao AWS KMS
- **Ledger** via `--ledger` flag (para deploy manual apenas)

```yaml
- name: Deploy with AWS KMS
  run: |
    # Assina transação usando AWS KMS
    forge script Deploy.s.sol \
      --rpc-url ${{ secrets.MAINNET_RPC_URL }} \
      --aws-kms-key-id ${{ secrets.KMS_KEY_ID }}
```

---

## 8. Multi-Environment Strategy

### Environments:

| Environment | Branch | Network | Purpose | Auto-deploy? |
|-------------|--------|---------|---------|--------------|
| **Development** | `feature/*` | Localhost | Dev local | Não |
| **Staging** | `develop` | Sepolia | Testing integrado | Sim |
| **Production** | `main` | Mainnet | Produção | Manual (approval) |

### Directory structure:

```
script/
├── Deploy.s.sol              # Deploy script principal
├── config/
│   ├── localhost.json        # Anvil local
│   ├── sepolia.json          # Testnet
│   └── mainnet.json          # Mainnet
└── utils/
    └── ConfigLoader.sol      # Helper para ler config

deployments/
├── localhost/
│   └── latest.json
├── sepolia/
│   └── latest.json
└── mainnet/
    └── latest.json           # NUNCA commitar este (tem addresses reais)
```

### Config-based deployment:

```json
// script/config/sepolia.json
{
  "tokenName": "My Token",
  "tokenSymbol": "MTK",
  "initialSupply": "1000000",
  "owner": "0x1234...",  // Multi-sig no Sepolia
  "minDelay": 3600       // 1 hour timelock
}
```

```solidity
// script/Deploy.s.sol
import {Script} from "forge-std/Script.sol";
import {MyToken} from "../src/MyToken.sol";
import {Timelock} from "../src/Timelock.sol";

contract DeployScript is Script {
    function run() external {
        // Lê config baseado na chain ID
        string memory configPath = string.concat(
            "script/config/",
            getChainName(block.chainid),
            ".json"
        );

        string memory json = vm.readFile(configPath);

        string memory tokenName = vm.parseJsonString(json, ".tokenName");
        string memory tokenSymbol = vm.parseJsonString(json, ".tokenSymbol");
        uint256 initialSupply = vm.parseJsonUint(json, ".initialSupply");
        address owner = vm.parseJsonAddress(json, ".owner");
        uint256 minDelay = vm.parseJsonUint(json, ".minDelay");

        uint256 deployerPrivateKey = vm.envUint("DEPLOYER_PRIVATE_KEY");

        vm.startBroadcast(deployerPrivateKey);

        MyToken token = new MyToken(tokenName, tokenSymbol, initialSupply);
        Timelock timelock = new Timelock(minDelay, owner);

        // Transfer ownership para timelock
        token.transferOwnership(address(timelock));

        vm.stopBroadcast();

        // Save deployment addresses
        string memory deployment = string.concat(
            '{"token":"', vm.toString(address(token)), '",',
            '"timelock":"', vm.toString(address(timelock)), '"}'
        );

        vm.writeFile(
            string.concat("deployments/", getChainName(block.chainid), "/latest.json"),
            deployment
        );
    }

    function getChainName(uint256 chainId) internal pure returns (string memory) {
        if (chainId == 1) return "mainnet";
        if (chainId == 11155111) return "sepolia";
        if (chainId == 31337) return "localhost";
        revert("Unsupported chain");
    }
}
```

---

## 9. Contract Verification

### Etherscan Verification (Foundry)

```yaml
- name: Deploy and verify
  run: |
    forge script script/Deploy.s.sol \
      --rpc-url ${{ secrets.SEPOLIA_RPC_URL }} \
      --private-key ${{ secrets.DEPLOYER_PRIVATE_KEY }} \
      --broadcast \
      --verify \
      --etherscan-api-key ${{ secrets.ETHERSCAN_API_KEY }}
```

**Se verificação falhar, rodar manualmente:**

```bash
forge verify-contract \
  --chain-id 11155111 \
  --compiler-version v0.8.20 \
  --constructor-args $(cast abi-encode "constructor(string,string)" "My Token" "MTK") \
  0x1234... \  # Contract address
  src/MyToken.sol:MyToken \
  --etherscan-api-key $ETHERSCAN_API_KEY
```

### Multi-chain Verification

```yaml
- name: Verify on multiple chains
  run: |
    # Ethereum
    forge verify-contract 0x1234... src/MyToken.sol:MyToken \
      --chain mainnet \
      --etherscan-api-key ${{ secrets.ETHERSCAN_API_KEY }}

    # Arbitrum
    forge verify-contract 0x1234... src/MyToken.sol:MyToken \
      --chain arbitrum \
      --etherscan-api-key ${{ secrets.ARBISCAN_API_KEY }}

    # Optimism
    forge verify-contract 0x1234... src/MyToken.sol:MyToken \
      --chain optimism \
      --etherscan-api-key ${{ secrets.OPTIMISTIC_ETHERSCAN_API_KEY }}
```

### Tenderly Verification (para debugging)

```yaml
- name: Verify on Tenderly
  run: |
    tenderly contracts verify \
      --networks sepolia \
      MyToken=0x1234...
  env:
    TENDERLY_ACCESS_KEY: ${{ secrets.TENDERLY_ACCESS_KEY }}
    TENDERLY_PROJECT_SLUG: my-project
```

---

## 10. Monitoring Integration

### Post-Deployment: Configurar Tenderly Alerts

```yaml
- name: Setup Tenderly monitoring
  run: |
    curl -X POST https://api.tenderly.co/api/v1/account/me/project/${{ secrets.TENDERLY_PROJECT }}/alerts \
      -H "X-Access-Key: ${{ secrets.TENDERLY_ACCESS_KEY }}" \
      -H "Content-Type: application/json" \
      -d '{
        "name": "Large Withdrawal",
        "network_id": "1",
        "contract_address": "${{ env.VAULT_ADDRESS }}",
        "alert_type": "event",
        "event_conditions": {
          "event_signature": "Withdrawn(address,uint256)",
          "filters": [
            {"parameter": "amount", "operator": "gte", "value": "100000000000000000000"}
          ]
        },
        "destinations": [
          {"type": "slack", "webhook_url": "${{ secrets.SLACK_WEBHOOK }}"}
        ]
      }'
```

### OpenZeppelin Defender Integration

```javascript
// script/defender-setup.js
const { AdminClient } = require('defender-admin-client');

async function setupDefender() {
  const client = new AdminClient({
    apiKey: process.env.DEFENDER_API_KEY,
    apiSecret: process.env.DEFENDER_API_SECRET
  });

  // Add contract
  await client.addContract({
    network: 'sepolia',
    address: process.env.CONTRACT_ADDRESS,
    name: 'MyVault',
    abi: require('../out/MyVault.sol/MyVault.json').abi
  });

  // Create Autotask (monitoring)
  await client.createAutotask({
    name: 'Check Vault Health',
    encodedZippedCode: await client.getEncodedZippedCodeFromFolder('./autotasks'),
    trigger: {
      type: 'schedule',
      frequencyMinutes: 60
    },
    paused: false
  });

  console.log('✅ Defender configured');
}

setupDefender().catch(console.error);
```

```yaml
- name: Configure OpenZeppelin Defender
  run: |
    npm install defender-admin-client
    node script/defender-setup.js
  env:
    DEFENDER_API_KEY: ${{ secrets.DEFENDER_API_KEY }}
    DEFENDER_API_SECRET: ${{ secrets.DEFENDER_API_SECRET }}
    CONTRACT_ADDRESS: ${{ env.VAULT_ADDRESS }}
```

---

## 11. Best Practices

### Checklist de CI/CD

**Antes de merge para `main`:**
- [ ] ✅ Todos os testes passando (unit, integration, fork, fuzz)
- [ ] ✅ Coverage >= 90%
- [ ] ✅ Slither sem issues de alta severidade
- [ ] ✅ Gas report reviewed (sem aumentos inesperados)
- [ ] ✅ Contract sizes < 24KB
- [ ] ✅ Solhint sem warnings críticos
- [ ] ✅ Code review aprovado (mínimo 2 pessoas)

**Antes de deploy para Sepolia:**
- [ ] ✅ Todos os checks acima
- [ ] ✅ Mythril scan completo
- [ ] ✅ Deployment script testado localmente (anvil)
- [ ] ✅ Config para Sepolia validado

**Antes de deploy para Mainnet:**
- [ ] ✅ Deploy e testes completos na Sepolia
- [ ] ✅ Audit profissional completo
- [ ] ✅ Timelock configurado (mínimo 2 dias)
- [ ] ✅ Multi-sig como owner (3/5 ou 4/7)
- [ ] ✅ Emergency pause mechanism testado
- [ ] ✅ Monitoring e alertas configurados
- [ ] ✅ Runbook de incident response pronto
- [ ] ✅ Bug bounty ativo
- [ ] ✅ Insurance/cobertura considerada

### Branch Protection Rules

Settings → Branches → Add rule para `main`:

```yaml
Branch protection rules:
✅ Require a pull request before merging
  ✅ Require approvals: 2
  ✅ Dismiss stale reviews
  ✅ Require review from Code Owners
✅ Require status checks to pass before merging
  ✅ Require branches to be up to date
  Status checks:
    - test (all variants)
    - gas-report
    - slither
    - coverage
    - lint
✅ Require conversation resolution before merging
✅ Do not allow bypassing the above settings (nem admins)
```

### Deployment Checklist Template

```markdown
## Pre-Deployment Checklist

### Code Quality
- [ ] All tests passing (link CI run)
- [ ] Coverage >= 90% (link Codecov)
- [ ] No Slither high/medium issues
- [ ] Gas costs reviewed and approved
- [ ] Code review by 2+ engineers

### Security
- [ ] Audit completed (link report)
- [ ] All audit issues resolved
- [ ] Slither, Mythril, Echidna scans clean
- [ ] Security docs updated

### Infrastructure
- [ ] Multi-sig wallet configured (addresses: ___)
- [ ] Timelock deployed (delay: ___ hours)
- [ ] Emergency pause tested
- [ ] Upgrade mechanism tested (if applicable)

### Monitoring
- [ ] Tenderly alerts configured
- [ ] OpenZeppelin Defender setup
- [ ] Slack/Discord notifications active
- [ ] On-call rotation scheduled

### Documentation
- [ ] README updated with contract addresses
- [ ] Security docs published
- [ ] User documentation ready
- [ ] Incident response runbook ready

### Deployment
- [ ] Testnet deployment successful (link)
- [ ] Testnet verified on Etherscan
- [ ] Configuration files reviewed
- [ ] Deployer wallet funded (gas)
- [ ] Deployment script dry-run completed

### Post-Deployment
- [ ] Contracts verified on Etherscan
- [ ] Ownership transferred to multi-sig
- [ ] Initial configuration completed
- [ ] Monitoring confirmed working
- [ ] Announcement prepared (blog/Twitter)

**Signed off by:**
- [ ] Lead Engineer: ___
- [ ] Security Lead: ___
- [ ] CTO: ___
```

---

## 12. Glossário

**Termos Web3 de DevOps:**

| Termo | Definição | Comparação Web2 |
|-------|-----------|-----------------|
| **Fork testing** | Testar contra state real de blockchain (clonar mainnet) | Integration testing com prod database |
| **Gas reporting** | Medir custo em gas de cada função | Performance profiling |
| **Slither** | Static analyzer para Solidity | ESLint/SonarQube |
| **Mythril** | Symbolic execution para encontrar bugs | Fuzzing + static analysis |
| **Contract verification** | Publicar source code no Etherscan para transparência | Source maps (não tem equivalente direto) |
| **Timelock** | Delay obrigatório antes de executar admin functions | Deployment approval workflow |
| **Multi-sig** | Múltiplas assinaturas necessárias para transação crítica | 2FA/MFA para produção |
| **Chain ID** | Identificador único da network (1=mainnet, 11155111=Sepolia) | Environment (dev/staging/prod) |
| **Broadcast** | Enviar transação assinada para blockchain | Deploy to server |
| **Deterministic deployment** | Mesmo bytecode → mesmo address em todas as chains | Reproducible builds |

**Ferramentas mencionadas:**
- **Foundry**: Toolkit Solidity (forge, cast, anvil)
- **Hardhat**: Framework JavaScript para Ethereum
- **Slither**: Static analyzer (Trail of Bits)
- **Mythril**: Symbolic execution engine (ConsenSys)
- **Tenderly**: Monitoring e debugging platform
- **OpenZeppelin Defender**: Security automation platform
- **Codecov**: Code coverage reporting

---

## 13. Exercícios

### Exercício 1: Setup CI Básico

**Objetivo**: Configurar CI completo para um projeto Foundry.

**Tarefa**:
1. Criar repo com projeto Foundry (token ERC-20 simples)
2. Adicionar workflow para testes
3. Adicionar gas reporting
4. Configurar Codecov
5. Adicionar Slither

**Critérios de sucesso**:
- ✅ Todos os workflows rodando
- ✅ PR comments com gas report
- ✅ Coverage badge no README
- ✅ Slither sem issues críticos

### Exercício 2: Multi-Environment Deployment

**Objetivo**: Criar sistema de deploy para localhost → Sepolia → Mainnet.

**Tarefa**:
1. Criar 3 configs (localhost.json, sepolia.json, mainnet.json)
2. Script de deploy que lê config baseado em chain ID
3. Workflow para deploy Sepolia (automático em `develop`)
4. Workflow para deploy Mainnet (manual approval)

**Critérios de sucesso**:
- ✅ Deploy local funciona com `forge script`
- ✅ Deploy Sepolia automático via CI
- ✅ Deploy Mainnet requer approval
- ✅ Deployment addresses salvos em `deployments/`

### Exercício 3: Security Pipeline

**Objetivo**: Pipeline completo de security scanning.

**Tarefa**:
1. Adicionar Slither ao CI (fail em high severity)
2. Adicionar Mythril para contratos críticos
3. Adicionar Solhint com regras customizadas
4. Criar script para rodar tudo localmente

**Critérios de sucesso**:
- ✅ Slither roda em cada PR
- ✅ Mythril roda antes de deploy
- ✅ Solhint enforça style guide
- ✅ Script `make security-check` roda tudo

### Solução Exercício 1

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Run tests
        run: forge test -vvv

      - name: Run coverage
        run: forge coverage --report lcov

      - name: Upload to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./lcov.info
          token: ${{ secrets.CODECOV_TOKEN }}

  gas-report:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Generate gas report
        run: forge test --gas-report > gas-report.txt

      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('gas-report.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## ⛽ Gas Report\n\`\`\`\n${report}\n\`\`\``
            });

  slither:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - run: pip3 install slither-analyzer

      - uses: foundry-rs/foundry-toolchain@v1

      - name: Run Slither
        run: slither . --exclude-dependencies --fail-on high
```

---

## 14. Recursos

### GitHub Actions para Blockchain

**Official Actions:**
- [foundry-toolchain](https://github.com/foundry-rs/foundry-toolchain) - Instalar Foundry
- [setup-node](https://github.com/actions/setup-node) - Para Hardhat
- [github-script](https://github.com/actions/github-script) - Comentar PRs

**Templates:**
- [foundry-template](https://github.com/foundry-rs/forge-template) - Template oficial Foundry
- [hardhat-template](https://github.com/NomicFoundation/hardhat-boilerplate) - Template Hardhat

### Security Tools

**Static Analysis:**
- [Slither](https://github.com/crytic/slither) - Trail of Bits
- [Mythril](https://github.com/ConsenSys/mythril) - ConsenSys
- [Aderyn](https://github.com/Cyfrin/aderyn) - Rust-based (rápido)

**Fuzzing:**
- [Echidna](https://github.com/crytic/echidna) - Property-based fuzzing
- [Foundry Fuzz](https://book.getfoundry.sh/forge/fuzz-testing) - Built-in

**Formal Verification:**
- [Certora](https://www.certora.com/) - Prover comercial
- [Halmos](https://github.com/a16z/halmos) - Symbolic testing (a16z)

### Monitoring & Automation

**Platforms:**
- [Tenderly](https://tenderly.co/) - Monitoring, debugging, simulations
- [OpenZeppelin Defender](https://defender.openzeppelin.com/) - Security automation
- [Gelato](https://www.gelato.network/) - Smart contract automation

**Analytics:**
- [Dune Analytics](https://dune.com/) - On-chain data dashboards
- [The Graph](https://thegraph.com/) - Indexing protocol

### Learning Resources

**Guides:**
- [Foundry Book - CI/CD](https://book.getfoundry.sh/tutorials/best-practices#ci-cd)
- [Hardhat - Deployment](https://hardhat.org/hardhat-runner/docs/guides/deploying)
- [Smart Contract Security Field Guide](https://scsfg.io/)

**Exemplos de Repos:**
- [OpenZeppelin Contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) - CI exemplar
- [Uniswap v4](https://github.com/Uniswap/v4-core) - DevOps profissional
- [Aave v3](https://github.com/aave/aave-v3-core) - Multi-chain deployment

---

## 15. Próximos Passos

### Depois de Dominar DevOps

**1. Continuous Security**
- Integrar audit automation (Certora Prover no CI)
- Bug bounty automation (Immunefi integration)
- Incident response automation

**2. Multi-Chain DevOps**
- Deploy para múltiplas chains (Ethereum, Arbitrum, Optimism, Polygon)
- Cross-chain verification
- Unified monitoring para todas as chains

**3. Advanced Deployment Strategies**
- Blue-green deployment (via proxy)
- Canary releases (limitar initial supply/users)
- Feature flags on-chain

**4. Monitoring Avançado**
- Machine learning para detectar anomalias
- Predictive alerts (antes de problemas acontecerem)
- Integration com PagerDuty/OpsGenie

### Checklist: Você Domina DevOps?

- [ ] Tem CI/CD configurado com testes, gas report, security scan
- [ ] Deploy automatizado para testnet
- [ ] Deploy para mainnet com approval e timelock
- [ ] Secrets management seguro (nunca comitou private key)
- [ ] Contract verification automática
- [ ] Monitoring e alertas configurados
- [ ] Incident response runbook documentado
- [ ] Multi-environment strategy (dev/staging/prod)

### Próximo Capítulo

**Cap 17: Auditoria e Bug Bounties**
- Como preparar contrato para audit
- Escolher firma de auditoria
- Configurar bug bounty (Immunefi, Code4rena)
- Formal verification com Certora
- Post-audit fixes

**Cap 18: Deployment e Upgrade Strategies**
- Phased rollout (começar pequeno)
- Emergency procedures
- Upgrade via proxy (quando e como)
- Mainnet launch checklist

---

## Conclusão

**DevOps para smart contracts != DevOps Web2.**

**Diferenças críticas:**
1. **Immutability**: Deploy errado = dinheiro perdido
2. **Cost**: Cada transação custa gas
3. **Security**: Vulnerabilidades podem drenar fundos
4. **Transparency**: Todo código é público (quando verified)

**Pipeline mínimo necessário:**
```
Code → Tests (unit/integration/fork/fuzz) → Security scan (Slither)
  → Gas report → Coverage → Deploy testnet → Monitoring → Mainnet (approval)
```

**Próxima vez que você criar um PR:**
1. ✅ CI roda testes automaticamente
2. ✅ Bot comenta gas report
3. ✅ Slither verifica vulnerabilidades
4. ✅ Coverage badge atualiza
5. ✅ Merge → deploy automático para Sepolia
6. ✅ Main → manual approval → deploy Mainnet com timelock

**Lembre-se:**
- 🔒 Nunca commite private keys
- 🧪 Testa tudo (unit/integration/fork/fuzz)
- 🔍 Security scanning em cada PR
- ⛽ Monitora gas costs
- 🚀 Deploy testnet primeiro, sempre
- ⏱️ Timelock para mainnet (mínimo 24h)
- 🔔 Alertas configurados antes de mainnet
- 📚 Runbook pronto para incidents

**DevOps não é opcional em blockchain - é questão de sobrevivência do protocolo.**

Um erro de deploy pode custar milhões. Um pipeline bem configurado salva vidas (e dinheiro).

---

**Você está pronto para produção quando seu CI rejeitar código ruim automaticamente.**

**Próximo capítulo**: Auditoria profissional e bug bounties. 🔐
