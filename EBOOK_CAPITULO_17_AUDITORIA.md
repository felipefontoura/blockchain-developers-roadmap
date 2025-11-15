# Capítulo 17: Auditoria e Segurança Avançada

> **Para Desenvolvedores Experientes**: Code review em Web2? 30 minutos, pior caso perde alguns dados. Auditoria em blockchain? Semanas de análise, pior caso perde $100M. Não existe "hotfix" - código é imutável. Auditoria não é luxo, é **obrigação** antes de mainnet. Este capítulo pode salvar milhões.

---

## Índice
- [17.1 Por Que Auditoria É Obrigatória](#171-por-que-auditoria-é-obrigatória)
- [17.2 Self-Audit Checklist](#172-self-audit-checklist)
- [17.3 Ferramentas Automatizadas](#173-ferramentas-automatizadas)
- [17.4 Auditoria Manual](#174-auditoria-manual)
- [17.5 Auditoria Profissional](#175-auditoria-profissional)
- [17.6 Bug Bounty Programs](#176-bug-bounty-programs)
- [17.7 Formal Verification](#177-formal-verification)
- [17.8 Post-Audit](#178-post-audit)

---

## 17.1 Por Que Auditoria É Obrigatória

### Custo de Bugs em Blockchain

**Web2**:
```
Bug encontrado em produção:
1. Hotfix em 10 minutos ✅
2. Rollback database ✅
3. Pedir desculpas aos users ✅
4. Ninguém perdeu dinheiro ✅
```

**Web3**:
```
Bug encontrado em produção:
1. ❌ Não pode fazer hotfix (imutável)
2. ❌ Não pode fazer rollback
3. ❌ $100M roubados (irreversível)
4. ❌ Projeto morto, carreira destruída
```

### Hacks Reais

| Projeto | Valor Perdido | Vulnerabilidade | Ano |
|---------|---------------|-----------------|-----|
| **Poly Network** | $600M | Access control | 2021 |
| **Ronin Bridge** | $625M | Compromised keys | 2022 |
| **Wormhole** | $325M | Signature verification | 2022 |
| **Nomad Bridge** | $190M | Initialization bug | 2022 |
| **Euler Finance** | $197M | Donation attack | 2023 |
| **Parity Multisig** | $150M | delegatecall bug | 2017 |
| **The DAO** | $60M | Reentrancy | 2016 |

**Total roubado em 2022**: **$3.8 bilhões**

💀 **Todos esses hacks eram evitáveis com auditoria adequada.**

### Níveis de Auditoria

| Nível | Descrição | Custo | TVL Recomendado |
|-------|-----------|-------|-----------------|
| **Self-Audit** | Você mesmo + ferramentas | Grátis | < $10k |
| **Community Review** | GitHub, forums | Grátis | < $50k |
| **Junior Auditor** | Freelancer, Code4rena | $5-20k | < $500k |
| **Audit Firm** | Trail of Bits, OpenZeppelin | $50-200k+ | $1M+ |
| **Multiple Audits** | 2-3 firms independentes | $150-500k+ | $10M+ |
| **Formal Verification** | Matemática provável | $100-300k+ | $100M+ |

---

## 17.2 Self-Audit Checklist

### Antes de Começar

- [ ] **Todos** os testes passam (100%)
- [ ] Coverage >= 95%
- [ ] Fuzzing rodou com 10k+ runs
- [ ] Deploy em testnet funcionou
- [ ] Código revisado por pelo menos 1 dev experiente

### Checklist Geral

#### Access Control
- [ ] Todas funções sensíveis têm modifier de acesso (`onlyOwner`, `onlyRole`)
- [ ] `onlyOwner` não é único ponto de falha (considere multi-sig)
- [ ] Funções `public` deveriam ser `external`?
- [ ] Funções `external` deveriam ser `internal`?
- [ ] Constructor inicializa corretamente (especialmente em upgradeable)

#### Reentrancy
- [ ] Seguir **Checks-Effects-Interactions** em TODAS funções
- [ ] Usar `ReentrancyGuard` da OpenZeppelin
- [ ] Atualizar state ANTES de external calls
- [ ] Cuidado com `call`, `delegatecall`, `transfer`

#### Integer Math
- [ ] Solidity 0.8+ (overflow checks automáticos)
- [ ] `unchecked` apenas onde GARANTIDO safe
- [ ] Cuidado com divisão (arredondamento)
- [ ] Operações de tempo não overflow (timestamps)

#### External Calls
- [ ] Verificar retorno de `call`, `delegatecall`
- [ ] `transfer` e `send` podem falhar (usar `call`)
- [ ] Loops com external calls = DoS risk
- [ ] Confiar apenas em contratos verificados

#### Gas & DoS
- [ ] Sem loops unbounded (limite máximo)
- [ ] Funções `view`/`pure` realmente são?
- [ ] Gas estimation testada
- [ ] Sem operações muito caras (evitar user DoS)

#### Oracle & Price Manipulation
- [ ] NUNCA usar preço spot de único DEX
- [ ] Usar Chainlink ou TWAP
- [ ] Verificar staleness de oracle data
- [ ] Circuit breakers para mudanças abruptas

#### Tokens & Transferências
- [ ] Verificar balances antes/depois de transfer
- [ ] Alguns tokens cobram fee (accounting correto?)
- [ ] Alguns tokens são upgradeable (risk?)
- [ ] Approve/TransferFrom: verificar allowance

#### Randomness
- [ ] NUNCA usar `block.timestamp`, `blockhash` para random
- [ ] Usar Chainlink VRF
- [ ] Commit-Reveal se VRF muito caro

#### Upgradeable Contracts
- [ ] Storage layout compatível entre versões
- [ ] Initializer protegido contra re-inicialização
- [ ] `_disableInitializers()` em constructor
- [ ] Timelock para upgrades (48h+)

#### Events & Logging
- [ ] Eventos para TODAS state changes
- [ ] Indexed parameters corretos
- [ ] Não emitir informação sensível

#### Testing
- [ ] Unit tests para CADA função pública
- [ ] Integration tests para fluxos completos
- [ ] Fork tests contra protocolos reais
- [ ] Edge cases testados (0, max, overflow)
- [ ] Reentrancy attack testado

---

## 17.3 Ferramentas Automatizadas

### 1. Slither (Trail of Bits)

**Melhor ferramenta estática** para Solidity.

```bash
# Install
pip3 install slither-analyzer

# Run
slither . --exclude-dependencies

# Specific checks
slither . --detect reentrancy-eth
slither . --detect uninitialized-state
```

**Detecta**:
- Reentrancy (91 variantes!)
- Uninitialized variables
- Dangerous delegatecall
- tx.origin usage
- Unused variables
- Solc version issues
- +70 detectores

**Output**:
```
MyContract.withdraw() (contracts/MyContract.sol#42-50) ignores return value by token.transfer(msg.sender,amount) (contracts/MyContract.sol#48)
Reference: https://github.com/crytic/slither/wiki/Detector-Documentation#unchecked-transfer
```

### 2. Mythril (ConsenSys)

**Symbolic execution** - testa TODOS caminhos possíveis.

```bash
# Install
pip3 install mythril

# Run
myth analyze contracts/MyContract.sol

# With timeout
myth analyze contracts/MyContract.sol --execution-timeout 300
```

**Detecta**:
- Reentrancy
- Integer overflow
- Unchecked call returns
- Delegatecall to untrusted
- Access control issues

**Slow** mas muito poderoso (minutos a horas).

### 3. Echidna (Trail of Bits)

**Fuzzer baseado em propriedades**.

```solidity
// TestContract.sol
contract TestMyContract {
    MyContract c;

    constructor() {
        c = new MyContract();
    }

    // Propriedade: balance nunca deve diminuir sozinho
    function echidna_balance_never_decreases() public view returns (bool) {
        return c.balanceOf(address(this)) >= 0;
    }
}
```

```bash
# Install
docker pull trailofbits/echidna

# Run
echidna-test contracts/TestMyContract.sol --contract TestMyContract
```

**Encontra**: Violações de invariantes através de fuzzing inteligente.

### 4. Foundry Built-in

```bash
# Invariant testing
forge test --match-test invariant

# Fork testing
forge test --fork-url $RPC_URL

# Gas report
forge test --gas-report

# Coverage
forge coverage
```

### 5. Aderyn (Cyfrin)

**Novo, focado em DeFi**.

```bash
# Install
cargo install aderyn

# Run
aderyn .
```

**Detecta**: Padrões específicos de DeFi (AMM bugs, oracle issues).

### Comparação de Ferramentas

| Ferramenta | Tipo | Speed | Precisão | Quando Usar |
|------------|------|-------|----------|-------------|
| **Slither** | Static | ⚡⚡⚡ Fast | Alta | Sempre! CI/CD |
| **Mythril** | Symbolic | 🐌 Slow | Muito Alta | Pré-deploy |
| **Echidna** | Fuzzing | ⚡⚡ Medium | Alta | Invariants |
| **Aderyn** | Static | ⚡⚡⚡ Fast | Média | DeFi |

💡 **Recomendação**: Rodar **TODOS** antes de deploy.

---

## 17.4 Auditoria Manual

### Processo de Auditoria Manual

**1. Entender o sistema**
- Ler documentação
- Desenhar arquitetura
- Identificar componentes críticos
- Listar assumptions

**2. Code review sistemático**
- Começar por funções `external` e `public`
- Seguir fluxo de dinheiro (ETH, tokens)
- Identificar state changes
- Verificar access control

**3. Threat modeling**
- Quem são os attackers?
- O que eles querem? (roubar, DoS, manipular)
- Quais são os vetores de ataque?

**4. Testar hipóteses**
- Escrever PoC de exploits
- Fuzzing direcionado
- Edge cases extremos

### Exemplo: Auditando um Vault

```solidity
contract Vault {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient balance");

        balances[msg.sender] -= amount;

        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

**Auditoria**:

✅ **Checks-Effects-Interactions**: OK (state update antes de call)

✅ **Reentrancy Guard**: Não precisa (CEI seguido)

✅ **Integer Overflow**: OK (Solidity 0.8+)

❌ **DoS via Revert**:
```solidity
// Attacker pode criar contrato que reverte em receive()
contract Attacker {
    receive() external payable {
        revert(); // Bloqueia withdraw!
    }
}
```

**Fix**: Pull pattern ou usar transfer com try/catch.

❌ **Missing Events**: Não emite eventos!

**Fix**:
```solidity
event Deposited(address indexed user, uint256 amount);
event Withdrawn(address indexed user, uint256 amount);
```

### Checklist de Code Review

**Para cada função**:
- [ ] Quem pode chamar? (access control)
- [ ] Que state muda? (effects)
- [ ] External calls? (interactions)
- [ ] Pode reverter? (DoS risk)
- [ ] Confia em input do user? (validation)
- [ ] Math correto? (overflow, division)
- [ ] Events emitidos?

**Para o contrato**:
- [ ] Initializer protegido?
- [ ] Pausable se necessário?
- [ ] Upgradeability planejada?
- [ ] Documentação clara?

---

## 17.5 Auditoria Profissional

### Principais Firmas de Auditoria

| Firma | Especialidade | Custo Estimado | Timeline |
|-------|---------------|----------------|----------|
| **Trail of Bits** | Security research, Slither | $50-100k+ | 4-8 semanas |
| **OpenZeppelin** | Smart contracts, biblioteca | $50-150k+ | 4-6 semanas |
| **ConsenSys Diligence** | Ethereum, Mythril | $40-80k+ | 4-6 semanas |
| **Certora** | Formal verification | $100-300k+ | 8-12 semanas |
| **Cyfrin (Pashov)** | DeFi, competitivo | $30-60k | 2-4 semanas |
| **Hacken** | Multi-chain | $20-50k | 2-4 semanas |

### Processo de Auditoria Profissional

**1. Preparação (1-2 semanas)**
- Freeze code (commit específico)
- Documentação completa
- Testes 100% coverage
- Deploy em testnet

**2. Kickoff Call**
- Explicar sistema aos auditores
- Identificar áreas de risco
- Definir scope

**3. Auditoria (2-6 semanas)**
- Review de código linha por linha
- Ferramentas automatizadas
- Exploit development
- Relatórios preliminares

**4. Relatório Final**
- Issues categorizados (Critical, High, Medium, Low, Informational)
- Recomendações de fix
- Prazo para resposta

**5. Remediation (1-2 semanas)**
- Fix todos issues Critical/High
- Fix a maioria dos Medium
- Re-audit de mudanças

**6. Publicação**
- Relatório público
- Badge de "Audited by X"

### Como Escolher Auditores

**Perguntas**:
- Auditaram projetos similares?
- Quem são os auditores? (LinkedIn, GitHub)
- Quantos auditores trabalharão? (mínimo 2)
- Qual metodologia? (ferramentas + manual)
- Incluem re-audit de fixes?
- Quanto custa?

**Red Flags**:
- ❌ Auditoria em 1 semana (muito rápido)
- ❌ Custo muito baixo (<$20k para projeto complexo)
- ❌ Não especificam quem audita
- ❌ Sem relatórios públicos anteriores
- ❌ "100% garantia de segurança"

### Lendo Relatório de Auditoria

**Severidade**:
- 🔴 **Critical**: Perda direta de fundos, exploit fácil
- 🟠 **High**: Perda de fundos em condições específicas
- 🟡 **Medium**: Comportamento incorreto, não perda direta
- 🔵 **Low**: Best practices, otimizações
- ⚪ **Informational**: Sugestões, typos

**O que procurar**:
- Total de issues (normal: 10-30)
- Quantos Critical/High? (ideal: 0 após fixes)
- Todos foram fixados?
- Re-audit confirmou fixes?

---

## 17.6 Bug Bounty Programs

### O Que É Bug Bounty

**Conceito**: Pagar hackers para encontrar bugs ANTES de malévolos.

**Plataformas**:
- **Immunefi** - maior para DeFi
- **Code4rena** - competitive audits
- **HackenProof**
- **Sherlock**

### Configurar Bug Bounty

```markdown
# Bug Bounty Program

## Rewards
- Critical: Up to $100,000
- High: Up to $50,000
- Medium: Up to $10,000
- Low: Up to $1,000

## Scope
In-scope:
- MainContract.sol (0x...)
- TokenContract.sol (0x...)

Out-of-scope:
- Frontend
- Known issues (see audit report)

## Severity Guidelines
Critical:
- Direct loss of funds
- Unauthorized state changes

High:
- Conditional loss of funds
- Griefing attacks

## Disclosure
- Private disclosure via email
- 90 days for fix before public disclosure
- Credit in acknowledgments
```

**Custo**: $10-50k reservados, paga apenas se encontram bugs.

**ROI**: Melhor que perder $100M!

### Immunefi Example

```
Project: Uniswap
Bounty: Up to $2,250,000
Critical bugs found: 0 (muito bem auditado!)

Project: Wormhole
Bounty: Up to $10,000,000
Result: Hackeado por $325M (bounty não foi suficiente incentivo)
```

---

## 17.7 Formal Verification

### O Que É

**Formal Verification = Prova matemática** que código é correto.

```
Teste tradicional:
- Testa 1000 casos
- Não garante que caso 1001 não tem bug

Formal Verification:
- Prova matematicamente que TODOS casos são corretos
- 100% garantia (se spec estiver correta)
```

### Certora Prover

**Ferramenta mais usada** para Solidity.

```solidity
// Specification (.spec file)
methods {
    function balanceOf(address) external returns (uint256) envfree;
    function totalSupply() external returns (uint256) envfree;
}

// Invariant: sum of balances = total supply
invariant sumOfBalancesEqualsTotalSupply()
    to_mathint(totalSupply()) == sumOfAllBalances();
```

```bash
# Run Certora
certoraRun MyToken.sol --verify MyToken:spec.spec
```

**Output**: PASS ou COUNTEREXAMPLE (com caso que quebra).

### Quando Usar

✅ **Use formal verification quando**:
- TVL > $100M
- Lógica matemática crítica (stablecoins, derivativos)
- Precisão é CRÍTICA
- Orçamento permite ($100k+)

❌ **Não use quando**:
- Projeto simples
- Budget limitado
- Deadline apertado

**Projetos que usam**:
- Maker DAO (DAI)
- Aave
- Compound
- Uniswap V3 (parcial)

---

## 17.8 Post-Audit

### Depois da Auditoria

**1. Fix TODOS Critical/High**
- Não fazer deploy com Critical unfixed
- High = muito arriscado também

**2. Considerar Medium/Low**
- Medium: fix a maioria
- Low: fix se fácil

**3. Re-audit**
- Novas mudanças precisam re-audit
- Mesmo que "pequenas" (podem introduzir bugs)

**4. Publicar Relatório**
- Transparência para comunidade
- "Audited by X" aumenta confiança

**5. Bug Bounty**
- Layer adicional de segurança
- Ongoing monitoring

**6. Monitoring**
- Tenderly alerts
- Defender auto-pause
- Multi-sig para admin

### Incident Response Plan

**Se bug encontrado EM PRODUÇÃO**:

```
1. PAUSE (se tiver pausable)
   - Imediato, não perguntar

2. ASSESS
   - Gravidade?
   - Quanto pode ser perdido?
   - Exploitable agora?

3. COMMUNICATE
   - Team interno: imediatamente
   - Community: honest update
   - "Investigating issue, contract paused"

4. FIX
   - Hotfix se upgradeable
   - Ou deploy novo contrato + migration

5. POST-MORTEM
   - O que aconteceu?
   - Por que auditoria não pegou?
   - Como prevenir futuro?
```

**Exemplo**: Euler Finance hack
- Bug encontrado: 5min depois exploited
- Pause NÃO existia
- $197M perdidos
- Lição: SEMPRE ter pause!

---

## 📖 Glossário

**Static Analysis**
> Análise de código sem executar.
> **Ferramentas**: Slither, Aderyn.

**Symbolic Execution**
> Executa código com valores simbólicos (não concretos).
> **Ferramentas**: Mythril, Certora.
> **Vantagem**: Testa TODOS caminhos.

**Fuzzing**
> Teste com inputs aleatórios/semi-aleatórios.
> **Ferramentas**: Echidna, Foundry.

**Formal Verification**
> Prova matemática de correção.
> **Ferramenta**: Certora Prover.

**Bug Bounty**
> Programa de recompensa por encontrar bugs.
> **Plataformas**: Immunefi, Code4rena.

**Severity Levels**
> Critical > High > Medium > Low > Informational

---

## 🔒 Pre-Deploy Checklist Final

**Code**:
- [ ] Slither passou (0 medium+)
- [ ] Mythril passou (0 issues)
- [ ] Echidna passou (10k+ runs)
- [ ] Coverage >= 95%
- [ ] Fork tests contra mainnet
- [ ] Auditoria profissional completa
- [ ] Todos Critical/High fixados
- [ ] Re-audit de fixes

**Infrastructure**:
- [ ] Multi-sig configurado (3-of-5 mínimo)
- [ ] Timelock implementado (48h+)
- [ ] Pausable onde apropriado
- [ ] Monitoring configurado (Tenderly/Defender)
- [ ] Bug bounty ativo

**Documentation**:
- [ ] Relatório de auditoria público
- [ ] README com contract addresses
- [ ] Architecture docs
- [ ] Incident response plan
- [ ] User guides

**Team**:
- [ ] Pelo menos 1 dev on-call 24/7
- [ ] Runbook para emergências
- [ ] Communication plan
- [ ] Insurance? (Nexus Mutual)

---

## 📝 Exercícios

### Exercício 1: Audit This

```solidity
contract VulnerableVault {
    mapping(address => uint256) public balances;
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 amount = balances[msg.sender];
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
        balances[msg.sender] = 0;
    }

    function emergencyWithdraw() external {
        require(msg.sender == owner);
        payable(owner).transfer(address(this).balance);
    }
}
```

**Tarefa**: Encontre TODAS vulnerabilidades. Escreva exploits.

<details>
<summary>💡 Dica</summary>

Procure por:
- Reentrancy
- Access control
- DoS
- Ordem de operações
</details>

---

## 📚 Recursos

### Ferramentas
1. **[Slither](https://github.com/crytic/slither)**
2. **[Mythril](https://github.com/ConsenSys/mythril)**
3. **[Echidna](https://github.com/crytic/echidna)**
4. **[Certora](https://www.certora.com/)**

### Audit Reports
- **[OpenZeppelin Audits](https://blog.openzeppelin.com/security-audits)**
- **[Trail of Bits Publications](https://github.com/trailofbits/publications)**
- **[Code4rena Reports](https://code4rena.com/reports)**

### Learn
- **[Secureum Bootcamp](https://secureum.substack.com/)**
- **[Smart Contract Security Field Guide](https://scsfg.io/)**

---

## 🎯 Conclusão

**Auditoria não é opcional. É OBRIGATÓRIO.**

Antes de mainnet:
1. ✅ Self-audit (este capítulo)
2. ✅ Ferramentas automatizadas (TODAS)
3. ✅ Auditoria profissional (>=1 firma)
4. ✅ Bug bounty (sempre ativo)
5. ✅ Monitoring 24/7

**Custo de auditoria**: $50-200k
**Custo de hack**: $1M-$600M

**Vale a pena? SIM.**

---

**Próximo**: [Capítulo 18 - Deployment](./EBOOK_CAPITULO_18_DEPLOYMENT.md)

---

**Autor**: Baseado em auditorias reais + hacks post-mortems
**Última Atualização**: 2025-11-14
