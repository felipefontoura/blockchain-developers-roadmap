# Capítulo 18: Deployment Strategies

> **Para Desenvolvedores Experientes**: Deploy em Web2? `git push heroku main` e done. Deploy em blockchain? Irreversível, público, imutável, e se foder = perde milhões. Não existe "rollback". Este capítulo é checklist de sobrevivência para mainnet.

---

## Índice
- [18.1 Testnet Deployment](#181-testnet-deployment)
- [18.2 Mainnet Checklist](#182-mainnet-checklist)
- [18.3 Multi-Sig Management](#183-multi-sig-management)
- [18.4 Timelock Implementation](#184-timelock-implementation)
- [18.5 Phased Rollout](#185-phased-rollout)
- [18.6 Contract Verification](#186-contract-verification)
- [18.7 Emergency Mechanisms](#187-emergency-mechanisms)
- [18.8 Post-Deployment](#188-post-deployment)

---

## 18.1 Testnet Deployment

### Por Que Testnet

**Nunca pule testnet.** Bugs que custam $0 em testnet custam $100M em mainnet.

**Testnets principais**:
| Network | Testnet | Faucet | Quando Usar |
|---------|---------|--------|-------------|
| **Ethereum** | Sepolia | [sepoliafaucet.com](https://sepoliafaucet.com) | Final testing |
| **Ethereum** | Goerli | Deprecated | Legado |
| **Arbitrum** | Arbitrum Sepolia | [faucet.quicknode.com](https://faucet.quicknode.com) | L2 testing |
| **Optimism** | OP Sepolia | [faucet.optimism.io](https://faucet.optimism.io) | L2 testing |
| **Polygon** | Mumbai | [faucet.polygon.technology](https://faucet.polygon.technology) | Sidechain |

### Deploy Script (Foundry)

```solidity
// script/Deploy.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "../src/MyToken.sol";
import "../src/MyStaking.sol";

contract DeployScript is Script {
    function run() external {
        // Load deployer private key from env
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");

        // Start broadcast (todas txs seguintes vão para chain)
        vm.startBroadcast(deployerPrivateKey);

        // 1. Deploy Token
        MyToken token = new MyToken("My Token", "MTK");
        console.log("Token deployed at:", address(token));

        // 2. Deploy Staking (depende de token)
        MyStaking staking = new MyStaking(address(token));
        console.log("Staking deployed at:", address(staking));

        // 3. Setup inicial
        token.mint(address(staking), 1_000_000 * 10**18); // Rewards pool
        console.log("Minted rewards to staking contract");

        // 4. Transfer ownership (se aplicável)
        // token.transferOwnership(MULTISIG_ADDRESS);

        vm.stopBroadcast();

        // Log addresses para salvar
        console.log("\n=== DEPLOYMENT SUMMARY ===");
        console.log("Network:", block.chainid);
        console.log("Token:", address(token));
        console.log("Staking:", address(staking));
        console.log("==========================\n");
    }
}
```

### Deploy em Testnet

```bash
# .env file
PRIVATE_KEY=0x...
SEPOLIA_RPC_URL=https://eth-sepolia.g.alchemy.com/v2/YOUR_KEY
ETHERSCAN_API_KEY=YOUR_KEY

# Source env
source .env

# Simulate primeiro (dry run)
forge script script/Deploy.s.sol:DeployScript --rpc-url $SEPOLIA_RPC_URL

# Deploy de verdade
forge script script/Deploy.s.sol:DeployScript \
  --rpc-url $SEPOLIA_RPC_URL \
  --broadcast \
  --verify \
  -vvvv

# Salvar deployment info
forge script script/Deploy.s.sol:DeployScript \
  --rpc-url $SEPOLIA_RPC_URL \
  --broadcast \
  --verify \
  | tee deployments/sepolia-$(date +%Y%m%d-%H%M%S).log
```

### Testnet Testing Checklist

- [ ] Deploy funciona sem erro
- [ ] Todos contratos verificados no Etherscan
- [ ] Funções `external` acessíveis via Etherscan
- [ ] Frontend conecta e lê dados
- [ ] Transações funcionam (approve, transfer, etc.)
- [ ] Events são emitidos corretamente
- [ ] Multi-sig controla admin functions (se aplicável)
- [ ] Gas costs são razoáveis
- [ ] Edge cases testados (0, max, etc.)

💡 **Dica**: Deixar no testnet por 1-2 semanas, testar com community.

---

## 18.2 Mainnet Checklist

### Pre-Deployment

**Security**:
- [ ] Auditoria completa por firma respeitável
- [ ] Todos Critical/High issues fixados
- [ ] Re-audit de fixes
- [ ] Slither/Mythril rodados (0 issues)
- [ ] Coverage >= 95%
- [ ] Fork tests contra mainnet

**Code**:
- [ ] Código final frozen (commit hash documentado)
- [ ] Nenhuma mudança após auditoria
- [ ] Compiler version fixado
- [ ] Optimizer configurado consistentemente
- [ ] Nenhum TODO/FIXME/XXX em código

**Infrastructure**:
- [ ] Multi-sig configurado (>=3-of-5)
- [ ] Timelock implementado (48h minimum)
- [ ] Pausable onde apropriado
- [ ] Emergency procedures documentadas
- [ ] Team on-call 24/7 primeira semana

**Documentation**:
- [ ] README completo
- [ ] Architecture diagram
- [ ] Audit report publicado
- [ ] Contract addresses documentadas
- [ ] User guides prontos

**Testing**:
- [ ] Testnet deployment successful
- [ ] Community testou por 1+ semana
- [ ] Frontend produção testado contra testnet
- [ ] Load testing (se aplicável)

**Legal/Compliance**:
- [ ] Terms of service
- [ ] Privacy policy
- [ ] Regulatory review (se necessário)
- [ ] Geographic restrictions (se aplicável)

### Deployment Day Checklist

**Manhã**:
- [ ] Team disponível (sem férias!)
- [ ] RPC provider confirmado (Alchemy/Infura)
- [ ] Gas price monitorado (esperar baixo se possível)
- [ ] Deployment script testado em testnet
- [ ] Backup plan se falhar

**Deploy**:
- [ ] Deploy contracts
- [ ] Verificar no Etherscan
- [ ] Inicializar contratos (se necessário)
- [ ] Configurar multi-sig como admin
- [ ] Renunciar ownership individual (se aplicável)
- [ ] Smoke tests via Etherscan

**Pós-Deploy Imediato**:
- [ ] Salvar todas addresses
- [ ] Verificar state inicial
- [ ] Testar frontend em produção
- [ ] Monitoring ativo (Tenderly/Defender)
- [ ] Bug bounty anunciado
- [ ] Comunicar à community

---

## 18.3 Multi-Sig Management

### Por Que Multi-Sig

**Single owner** = single point of failure

```solidity
// ❌ PERIGOSO
contract MyContract is Ownable {
    function pause() external onlyOwner {
        _pause();
    }
}
// Se owner private key vazar = game over
```

**Multi-sig** = n-of-m signatures required

```
3-of-5 Multi-sig:
- 5 signers
- 3 precisam aprovar
- Mais seguro (precisa comprometer 3)
```

### Gnosis Safe Setup

**1. Deploy Multi-Sig** (via [safe.global](https://safe.global))

```
Signers:
- 0xAlice (CEO)
- 0xBob (CTO)
- 0xCarol (Security Lead)
- 0xDave (Core Dev)
- 0xEve (Core Dev)

Threshold: 3-of-5

Address: 0x... (SALVAR)
```

**2. Transfer Ownership**

```solidity
// Após deploy em mainnet
MyContract contract = MyContract(CONTRACT_ADDRESS);

// Trocar owner para multi-sig
contract.transferOwnership(MULTISIG_ADDRESS);

// Renunciar ownership pessoal (irreversível!)
// Agora APENAS multi-sig pode admin
```

**3. Fazer Transação via Multi-Sig**

```
1. Alice propõe: "Pause contract"
   - Cria tx no Gnosis Safe UI
   - Assina (1/3)

2. Bob aprova:
   - Revisa tx
   - Assina (2/3)

3. Carol aprova:
   - Revisa tx
   - Assina (3/3)
   - TX EXECUTADA automaticamente
```

### Multi-Sig Best Practices

**Signers**:
- ✅ Mix de roles (tech + non-tech)
- ✅ Geographic diversity (diferentes timezones)
- ✅ Different hardware (Ledger, Trezor, etc.)
- ✅ Conhecidos e confiáveis
- ❌ Todos na mesma empresa/localização
- ❌ Threshold muito alto (5-of-5 = pode travararar se 1 perder chave)

**Threshold**:
- 2-of-3: Rápido, mas menos seguro
- 3-of-5: **Recomendado para produção**
- 4-of-7: Alta segurança, mais lento

**Process**:
- Documentar procedimentos
- Review obrigatório (não blind signing)
- Test em testnet primeiro
- Emergency fast-track (se pausable)

---

## 18.4 Timelock Implementation

### Por Que Timelock

**Problema**: Multi-sig pode fazer upgrade imediatamente → users não têm tempo de reagir.

**Solução**: **Timelock** = delay obrigatório entre proposta e execução.

```
User:
"Não gosto desse upgrade? Tenho 48h para sacar meus fundos antes!"
```

### Timelock Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Timelock {
    uint256 public constant DELAY = 2 days;
    address public admin;

    mapping(bytes32 => uint256) public queuedTransactions;

    event QueueTransaction(
        bytes32 indexed txHash,
        address indexed target,
        uint256 value,
        string signature,
        bytes data,
        uint256 eta
    );

    event ExecuteTransaction(
        bytes32 indexed txHash,
        address indexed target,
        uint256 value,
        string signature,
        bytes data
    );

    event CancelTransaction(bytes32 indexed txHash);

    constructor(address _admin) {
        admin = _admin; // Multi-sig address
    }

    modifier onlyAdmin() {
        require(msg.sender == admin, "Not admin");
        _;
    }

    function queueTransaction(
        address target,
        uint256 value,
        string memory signature,
        bytes memory data
    ) external onlyAdmin returns (bytes32) {
        bytes32 txHash = keccak256(abi.encode(target, value, signature, data, block.timestamp));
        uint256 eta = block.timestamp + DELAY;

        queuedTransactions[txHash] = eta;

        emit QueueTransaction(txHash, target, value, signature, data, eta);
        return txHash;
    }

    function executeTransaction(
        address target,
        uint256 value,
        string memory signature,
        bytes memory data
    ) external payable onlyAdmin returns (bytes memory) {
        bytes32 txHash = keccak256(abi.encode(target, value, signature, data, block.timestamp - DELAY));
        uint256 eta = queuedTransactions[txHash];

        require(eta != 0, "Transaction not queued");
        require(block.timestamp >= eta, "Timelock not expired");
        require(block.timestamp <= eta + 7 days, "Transaction stale");

        delete queuedTransactions[txHash];

        bytes memory callData;
        if (bytes(signature).length == 0) {
            callData = data;
        } else {
            callData = abi.encodePacked(bytes4(keccak256(bytes(signature))), data);
        }

        (bool success, bytes memory returnData) = target.call{value: value}(callData);
        require(success, "Transaction execution reverted");

        emit ExecuteTransaction(txHash, target, value, signature, data);
        return returnData;
    }

    function cancelTransaction(
        address target,
        uint256 value,
        string memory signature,
        bytes memory data
    ) external onlyAdmin {
        bytes32 txHash = keccak256(abi.encode(target, value, signature, data, block.timestamp - DELAY));
        delete queuedTransactions[txHash];
        emit CancelTransaction(txHash);
    }
}
```

### Usage

```solidity
// 1. Multi-sig queue upgrade
timelock.queueTransaction(
    proxyAddress,
    0,
    "upgradeTo(address)",
    abi.encode(newImplementation)
);

// 2. Wait 2 days (users podem sacar fundos se não gostarem)

// 3. Multi-sig executa
timelock.executeTransaction(
    proxyAddress,
    0,
    "upgradeTo(address)",
    abi.encode(newImplementation)
);
```

**Delays comuns**:
- Uniswap: 2 dias
- Compound: 2 dias
- Aave: 1 dia (short timelock) + 7 dias (long timelock)
- MakerDAO: Variable (governance voting period)

---

## 18.5 Phased Rollout

### Não Fazer "Big Bang" Launch

**❌ Bad**:
```
Day 1: Deploy
Day 1: Announce to everyone
Day 1: $100M TVL
Day 2: Bug found
Day 2: $100M lost
```

**✅ Good**:
```
Week 1: Deploy + cap at $10k TVL
Week 2: Increase cap to $100k
Week 3: Increase cap to $1M
Week 4: Remove cap (if no issues)
```

### Supply Cap Implementation

```solidity
contract CappedVault {
    uint256 public maxTVL = 10_000 * 10**18; // $10k
    address public owner;

    function deposit(uint256 amount) external {
        require(totalDeposits + amount <= maxTVL, "Cap reached");
        // ...
    }

    function increaseCap(uint256 newCap) external onlyOwner {
        require(newCap > maxTVL, "Can only increase");
        maxTVL = newCap;
    }

    function removeCap() external onlyOwner {
        maxTVL = type(uint256).max;
    }
}
```

### Phased Timeline

**Week 0: Mainnet Deploy**
- Cap: $10k TVL
- Team only (internal testing)
- Monitoring 24/7

**Week 1: Friends & Family**
- Cap: $50k TVL
- Whitelist (known addresses)
- Collect feedback

**Week 2: Limited Public**
- Cap: $500k TVL
- Public announcement (small community)
- Bug bounty active

**Week 3: Increased Cap**
- Cap: $5M TVL
- Marketing ramp up
- Multi-sig + timelock enforced

**Week 4+: Full Launch**
- Remove cap (or high cap like $100M)
- Major announcement
- CEX listings (if token)

**If bug found**: Pause, fix, re-audit, restart phases.

---

## 18.6 Contract Verification

### Por Que Verificar

**Unverified contract** = black box (users não confiam)

**Verified contract** = source code público (transparência)

### Verify com Foundry

```bash
# Durante deploy
forge script script/Deploy.s.sol:DeployScript \
  --rpc-url $MAINNET_RPC_URL \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_API_KEY

# Ou depois
forge verify-contract \
  --chain-id 1 \
  --num-of-optimizations 200 \
  --watch \
  --constructor-args $(cast abi-encode "constructor(string,string)" "MyToken" "MTK") \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  --compiler-version v0.8.20+commit.a1b79de6 \
  0xYourContractAddress \
  src/MyToken.sol:MyToken
```

### Verify Manually (Etherscan)

1. Ir para [etherscan.io/verifyContract](https://etherscan.io/verifyContract)
2. Enter contract address
3. Select compiler version (exact!)
4. Select optimizer (on/off + runs)
5. Paste source code (flattened)
6. Enter constructor arguments (ABI encoded)
7. Submit

**Flatten código**:
```bash
forge flatten src/MyContract.sol > flattened.sol
```

### Multi-File Verification

Para contratos com imports:

```bash
# Standard JSON Input
forge verify-contract \
  --chain-id 1 \
  --watch \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  0xYourContractAddress \
  src/MyToken.sol:MyToken \
  --via-ir # Se usa via-ir no foundry.toml
```

---

## 18.7 Emergency Mechanisms

### Pausable

```solidity
import "@openzeppelin/contracts/security/Pausable.sol";

contract EmergencyVault is Pausable {
    address public owner;

    function deposit() external payable whenNotPaused {
        // ...
    }

    function withdraw(uint256 amount) external whenNotPaused {
        // ...
    }

    // Emergency: pause tudo
    function pause() external onlyOwner {
        _pause();
    }

    function unpause() external onlyOwner {
        _unpause();
    }

    // Emergency: sacar tudo (apenas se pausado)
    function emergencyWithdrawAll() external onlyOwner whenPaused {
        payable(owner).transfer(address(this).balance);
    }
}
```

### Circuit Breaker com Rate Limit

```solidity
contract RateLimitedVault {
    uint256 public constant MAX_WITHDRAW_PER_HOUR = 100 ether;
    uint256 public withdrawnThisHour;
    uint256 public hourStart;

    function withdraw(uint256 amount) external {
        // Reset counter a cada hora
        if (block.timestamp >= hourStart + 1 hours) {
            withdrawnThisHour = 0;
            hourStart = block.timestamp;
        }

        require(withdrawnThisHour + amount <= MAX_WITHDRAW_PER_HOUR, "Rate limit");

        withdrawnThisHour += amount;

        // ... withdraw logic
    }
}
```

### Guardian Multi-Sig

**Conceito**: 2 multi-sigs
- **Admin Multi-sig** (3-of-5): Upgrades, param changes
- **Guardian Multi-sig** (2-of-3): **APENAS pause** (emergency)

```solidity
contract GuardedVault is Pausable {
    address public adminMultisig;
    address public guardianMultisig;

    modifier onlyAdmin() {
        require(msg.sender == adminMultisig);
        _;
    }

    modifier onlyGuardian() {
        require(msg.sender == guardianMultisig);
        _;
    }

    // Apenas guardian pode pausar (rápido em emergência)
    function pause() external onlyGuardian {
        _pause();
    }

    // Apenas admin pode unpause (depois de fix)
    function unpause() external onlyAdmin {
        _unpause();
    }
}
```

**Vantagem**: Guardian = threshold baixo (2-of-3) = rápido em emergência.

---

## 18.8 Post-Deployment

### Primeiras 24 Horas

- [ ] Monitoring ativo (Tenderly/Defender)
- [ ] Team on-call
- [ ] Verificar primeiras transações
- [ ] Gas costs razoáveis?
- [ ] Events emitidos corretamente?
- [ ] Frontend funcionando?

### Primeira Semana

- [ ] Bug bounty anunciado
- [ ] Community testando
- [ ] Nenhum issue crítico encontrado
- [ ] TVL crescendo organicamente
- [ ] Marketing moderado (não overhype)

### Primeiro Mês

- [ ] Auditoria contínua (monitoring)
- [ ] Community feedback incorporado
- [ ] Docs atualizadas
- [ ] Analytics dashboard público
- [ ] Planning próximas features

### Deployment Artifacts

**Salvar**:
```
deployments/
├── mainnet/
│   ├── 2024-11-14-deployment.json
│   ├── 2024-11-14-deployment.log
│   └── contracts/
│       ├── MyToken-0x123.json
│       └── MyStaking-0x456.json
└── testnet/
    └── ...
```

**JSON format**:
```json
{
  "network": "mainnet",
  "chainId": 1,
  "deployer": "0x...",
  "timestamp": "2024-11-14T10:00:00Z",
  "contracts": {
    "MyToken": {
      "address": "0x123...",
      "blockNumber": 18500000,
      "txHash": "0xabc...",
      "verified": true
    },
    "MyStaking": {
      "address": "0x456...",
      "blockNumber": 18500001,
      "txHash": "0xdef...",
      "verified": true
    }
  },
  "config": {
    "timelockDelay": "172800",
    "multisigAddress": "0x789...",
    "guardianAddress": "0x012..."
  }
}
```

---

## 📖 Glossário

**Multi-Sig**
> Carteira que requer múltiplas assinaturas (n-of-m).
> **Exemplo**: 3-of-5 = 3 de 5 devem assinar.

**Timelock**
> Delay obrigatório entre proposta e execução.
> **Típico**: 24-48 horas.

**Circuit Breaker**
> Mecanismo de pause em emergência.
> **Analogia**: Disjuntor elétrico.

**Phased Rollout**
> Launch gradual com caps crescentes.
> **Por que**: Limitar dano se bug.

**Guardian**
> Multi-sig com poder APENAS de pause.
> **Threshold**: Mais baixo que admin (2-of-3 vs 3-of-5).

**TVL (Total Value Locked)**
> Total de fundos depositados no protocol.

---

## 🔒 Pre-Mainnet Final Checklist

**Security** (Zero Tolerance):
- [ ] ✅ Auditoria profissional completa
- [ ] ✅ Todos Critical/High fixados
- [ ] ✅ Re-audit de fixes
- [ ] ✅ Slither/Mythril clean
- [ ] ✅ Coverage >= 95%
- [ ] ✅ Fork tests passando

**Infrastructure** (Fail-Safe):
- [ ] ✅ Multi-sig configurado (3-of-5+)
- [ ] ✅ Timelock (48h+)
- [ ] ✅ Guardian multi-sig (pause)
- [ ] ✅ Monitoring (Tenderly/Defender)
- [ ] ✅ Bug bounty ativo

**Process** (Emergency Ready):
- [ ] ✅ Incident response plan
- [ ] ✅ Team on-call 24/7 (primeira semana)
- [ ] ✅ Communication plan
- [ ] ✅ Pausable implemented
- [ ] ✅ Emergency withdrawal plan

**Rollout** (Conservative):
- [ ] ✅ Start com TVL cap ($10k-$50k)
- [ ] ✅ Phased increase over weeks
- [ ] ✅ Marketing moderado (não overhype)
- [ ] ✅ Community testing em testnet

**Documentation** (Transparency):
- [ ] ✅ README completo
- [ ] ✅ Audit report público
- [ ] ✅ Contract addresses documentadas
- [ ] ✅ User guides
- [ ] ✅ Architecture diagrams

---

## 📝 Deployment Runbook

### Day of Deployment

**08:00 - Team Meeting**
- Confirm go/no-go
- Review checklist
- Assign roles

**09:00 - Final Tests**
- Run deployment script em testnet
- Verify gas estimates
- Check RPC provider status

**10:00 - Deploy Mainnet**
```bash
# 1. Deploy contracts
forge script script/Deploy.s.sol --broadcast --verify

# 2. Verify deployment
# Check Etherscan

# 3. Initialize
# Via multi-sig

# 4. Transfer ownership
# To multi-sig + timelock
```

**11:00 - Verification**
- Contracts verified on Etherscan
- Frontend connects
- Smoke tests successful

**12:00 - Monitoring Setup**
- Tenderly alerts active
- Defender auto-tasks running
- Team ready on Discord

**13:00 - Announce**
- Twitter/Discord announcement
- Docs published
- Bug bounty live

**14:00-24:00 - Watch**
- Monitor first transactions
- Respond to community
- No major changes

---

## 📚 Recursos

### Tools
- **[Gnosis Safe](https://safe.global)** - Multi-sig
- **[OpenZeppelin Defender](https://defender.openzeppelin.com)** - Monitoring
- **[Tenderly](https://tenderly.co)** - Alerts

### Guides
- **[Deployment Best Practices](https://ethereum.org/en/developers/docs/deployment/)**

---

## 🎯 Conclusão

**Deployment não é fim - é INÍCIO.**

Post-deployment:
- Monitoring constante
- Bug bounty ativo
- Community feedback
- Phased growth
- Emergency ready

**Lembre**: Código é imutável. Deploy com cuidado.

---

**Próximo**: [Capítulo 19 - Monitoring](./EBOOK_CAPITULO_19_MONITORING.md)

---

**Autor**: Baseado em deployments reais + post-mortems
**Última Atualização**: 2025-11-14
