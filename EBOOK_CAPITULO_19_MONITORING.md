# Capítulo 19: Monitoring e Incident Response

> **Para Desenvolvedores Experientes**: Em Web2, Sentry/Datadog te avisam se server cai. Em blockchain, precisa monitorar $100M 24/7 - nenhum "server" para reiniciar, nenhum rollback. Monitoring não é opcional: é diferença entre detectar exploit em 5 minutos vs 5 horas (e perder tudo).

---

## Índice
- [19.1 O Que Monitorar](#191-o-que-monitorar)
- [19.2 Tenderly](#192-tenderly)
- [19.3 OpenZeppelin Defender](#193-openzeppelin-defender)
- [19.4 Alertas Críticos](#194-alertas-críticos)
- [19.5 Dashboards](#195-dashboards)
- [19.6 Incident Response](#196-incident-response)
- [19.7 Post-Mortem](#197-post-mortem)
- [19.8 Communication Strategy](#198-communication-strategy)

---

## 19.1 O Que Monitorar

### Categorias de Monitoring

| Categoria | O Que | Quando Alertar |
|-----------|-------|----------------|
| **Security** | Transações suspeitas, admin calls | Imediatamente |
| **Financial** | Large transfers, price anomalies | Imediatamente |
| **Operational** | Gas prices, failed txs | Moderado |
| **User Activity** | TVL, volume, users ativos | Daily report |
| **Health** | Oracle freshness, reserves | Hourly |

### Métricas Críticas

**Security**:
- ✅ Chamadas a funções admin (`pause`, `upgrade`, `transferOwnership`)
- ✅ Grandes saques (> $100k)
- ✅ Transações de addresses desconhecidos com muito gas
- ✅ Mudanças em parâmetros críticos

**Financial**:
- ✅ TVL (Total Value Locked)
- ✅ Volume diário
- ✅ Liquidações (em lending)
- ✅ Slippage anormal (em DEX)
- ✅ Divergência de preços (oracle vs market)

**Operational**:
- ✅ Gas usado por transação
- ✅ Taxa de falha de transações
- ✅ Queue size (para pending txs)
- ✅ RPC latency

**Health Checks**:
- ✅ Oracle timestamps (não stale)
- ✅ Reserves/colaterals suficientes
- ✅ Smart contract balances
- ✅ Multi-sig nonce (não travado)

---

## 19.2 Tenderly

### Setup

```bash
# Install CLI
npm install -g @tenderly/cli

# Login
tenderly login

# Init project
tenderly init
```

**tenderly.yaml**:
```yaml
account_id: "your-username"
project_slug: "your-project"
contracts:
  - name: MyToken
    address: "0x..."
    network: "1" # Mainnet
  - name: MyStaking
    address: "0x..."
    network: "1"
```

```bash
# Push
tenderly push
```

### Web3 Actions (Alertas)

**Alert quando admin function chamada**:

```javascript
// tenderly/actions/admin-call-alert.js
const { ethers } = require("ethers");
const axios = require("axios");

const ADMIN_FUNCTIONS = [
  "pause()",
  "unpause()",
  "upgradeTo(address)",
  "transferOwnership(address)",
  "setParameter(uint256)"
];

module.exports.actionFn = async (context, event) => {
  const tx = event.transaction;

  // Decode function call
  const iface = new ethers.Interface(context.metadata.getAbi());
  const decoded = iface.parseTransaction({ data: tx.input });

  // Check se é admin function
  if (ADMIN_FUNCTIONS.includes(decoded.signature)) {
    // Send alert
    await axios.post(process.env.DISCORD_WEBHOOK, {
      content: `🚨 **ADMIN CALL DETECTED**
Function: ${decoded.signature}
Caller: ${tx.from}
Tx: https://etherscan.io/tx/${tx.hash}
      `
    });

    await axios.post(process.env.TELEGRAM_BOT_URL, {
      chat_id: process.env.TELEGRAM_CHAT_ID,
      text: `🚨 Admin function called: ${decoded.signature} by ${tx.from}`
    });
  }
};

module.exports.trigger = {
  type: "transaction",
  transaction: {
    status: ["confirmed"],
    filters: [
      {
        network: "1",
        to: "0xYourContractAddress"
      }
    ]
  }
};
```

**Alert quando saque grande**:

```javascript
// tenderly/actions/large-withdrawal-alert.js
module.exports.actionFn = async (context, event) => {
  const LARGE_AMOUNT = ethers.parseEther("100"); // 100 ETH

  const logs = event.logs;

  for (const log of logs) {
    if (log.topics[0] === ethers.id("Withdrawal(address,uint256)")) {
      const amount = ethers.AbiCoder.defaultAbiCoder().decode(
        ["uint256"],
        log.data
      )[0];

      if (amount >= LARGE_AMOUNT) {
        // ALERT!
        await sendAlert(`Large withdrawal: ${ethers.formatEther(amount)} ETH`);
      }
    }
  }
};
```

### Simulation (Dry Run)

**Simular transação ANTES de enviar**:

```javascript
const axios = require('axios');

async function simulateTx(from, to, data, value = "0") {
  const response = await axios.post(
    'https://api.tenderly.co/api/v1/account/YOUR_ACCOUNT/project/YOUR_PROJECT/simulate',
    {
      network_id: "1",
      from,
      to,
      input: data,
      value,
      save: true, // Salvar simulação
      save_if_fails: true
    },
    {
      headers: {
        'X-Access-Key': process.env.TENDERLY_ACCESS_KEY
      }
    }
  );

  return response.data;
}

// Usage
const result = await simulateTx(
  "0xYourAddress",
  "0xContractAddress",
  "0x..." // encoded function call
);

if (!result.transaction.status) {
  console.error("Transaction will fail:", result.transaction.error_message);
} else {
  console.log("Gas used:", result.transaction.gas_used);
  console.log("State changes:", result.simulation.stateDiff);
}
```

---

## 19.3 OpenZeppelin Defender

### Sentinel (Alerts)

**1. Create Sentinel via UI** ([defender.openzeppelin.com](https://defender.openzeppelin.com))

**2. Or via code**:

```javascript
// defender/sentinels/admin-monitor.js
const { Defender } = require('@openzeppelin/defender-sdk');

const client = new Defender({
  apiKey: process.env.DEFENDER_API_KEY,
  apiSecret: process.env.DEFENDER_API_SECRET,
});

async function createSentinel() {
  const sentinel = await client.sentinels.create({
    type: 'BLOCK',
    name: 'Admin Function Monitor',
    addresses: ['0xYourContractAddress'],
    abi: require('../abi/MyContract.json'),
    paused: false,
    eventConditions: [],
    functionConditions: [
      {
        functionSignature: 'pause()',
      },
      {
        functionSignature: 'upgradeTo(address)',
      }
    ],
    txCondition: 'status == "success"',
    autotaskTrigger: 'YOUR_AUTOTASK_ID', // Autotask que envia alerta
  });

  console.log('Sentinel created:', sentinel.sentinelId);
}
```

### Autotask (Automated Actions)

**Auto-pause se exploit detectado**:

```javascript
// defender/autotasks/auto-pause.js
const { ethers } = require('ethers');
const { DefenderRelayProvider, DefenderRelaySigner } = require('@openzeppelin/defender-relay-client/lib/ethers');

exports.handler = async function(credentials, context) {
  const provider = new DefenderRelayProvider(credentials);
  const signer = new DefenderRelaySigner(credentials, provider, { speed: 'fast' });

  const CONTRACT_ADDRESS = '0x...';
  const ABI = [
    'function pause() external',
    'function isPaused() view returns (bool)'
  ];

  const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, signer);

  // Check se já pausado
  const isPaused = await contract.isPaused();
  if (isPaused) {
    console.log('Already paused');
    return;
  }

  // PAUSE IMEDIATAMENTE
  const tx = await contract.pause();
  console.log('Paused! Tx:', tx.hash);

  await tx.wait();

  // Send alerts
  await sendDiscordAlert('🚨 CONTRACT AUTO-PAUSED DUE TO SUSPICIOUS ACTIVITY');
  await sendTelegramAlert('Contract paused by Defender Autotask');

  return { paused: true, tx: tx.hash };
};
```

**Scheduled task** (daily health check):

```javascript
// defender/autotasks/daily-health-check.js
exports.handler = async function(credentials, context) {
  const provider = new DefenderRelayProvider(credentials);
  const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, provider);

  // Check metrics
  const tvl = await contract.getTotalValueLocked();
  const lastOracleUpdate = await contract.lastOracleUpdate();
  const reserves = await contract.getReserves();

  // Validate
  const issues = [];

  if (block.timestamp - lastOracleUpdate > 3600) {
    issues.push('Oracle stale (>1h)');
  }

  if (tvl === 0n) {
    issues.push('Zero TVL');
  }

  if (reserves < tvl) {
    issues.push('Insufficient reserves');
  }

  if (issues.length > 0) {
    await sendAlert(`❌ Health check failed:\n${issues.join('\n')}`);
  } else {
    console.log('✅ Health check passed');
  }

  return { healthy: issues.length === 0, issues };
};
```

---

## 19.4 Alertas Críticos

### Canais de Alerta

**Priority Levels**:

| Level | Canais | Response Time |
|-------|--------|---------------|
| **P0 - Critical** | PagerDuty + Phone + Telegram + Discord | 5 minutos |
| **P1 - High** | Telegram + Discord | 30 minutos |
| **P2 - Medium** | Discord | 2 horas |
| **P3 - Low** | Email | 24 horas |

### Discord Webhook

```javascript
async function sendDiscordAlert(message, priority = "HIGH") {
  const WEBHOOK_URL = process.env.DISCORD_WEBHOOK;

  const colors = {
    CRITICAL: 0xFF0000, // Red
    HIGH: 0xFFA500,     // Orange
    MEDIUM: 0xFFFF00,   // Yellow
    LOW: 0x00FF00       // Green
  };

  await axios.post(WEBHOOK_URL, {
    embeds: [{
      title: `🚨 ${priority} Alert`,
      description: message,
      color: colors[priority],
      timestamp: new Date().toISOString(),
      footer: {
        text: 'Blockchain Monitoring'
      }
    }]
  });
}

// Usage
await sendDiscordAlert(
  `Large withdrawal detected: ${amount} ETH from ${address}`,
  "CRITICAL"
);
```

### Telegram Bot

```javascript
async function sendTelegramAlert(message) {
  const BOT_TOKEN = process.env.TELEGRAM_BOT_TOKEN;
  const CHAT_ID = process.env.TELEGRAM_CHAT_ID;

  await axios.post(`https://api.telegram.org/bot${BOT_TOKEN}/sendMessage`, {
    chat_id: CHAT_ID,
    text: message,
    parse_mode: 'Markdown'
  });
}
```

### PagerDuty (P0 apenas)

```javascript
async function sendPagerDutyAlert(message, severity = "critical") {
  await axios.post('https://events.pagerduty.com/v2/enqueue', {
    routing_key: process.env.PAGERDUTY_INTEGRATION_KEY,
    event_action: 'trigger',
    payload: {
      summary: message,
      severity: severity,
      source: 'blockchain-monitoring',
      component: 'smart-contract',
      custom_details: {
        contract: CONTRACT_ADDRESS,
        network: 'mainnet'
      }
    }
  });
}
```

---

## 19.5 Dashboards

### Dune Analytics

**Create SQL query**:

```sql
-- Daily Volume
SELECT
  DATE_TRUNC('day', block_time) AS date,
  SUM(amount_usd) AS volume_usd,
  COUNT(*) AS num_trades
FROM dex.trades
WHERE project = 'YourProject'
  AND block_time > NOW() - INTERVAL '30 days'
GROUP BY 1
ORDER BY 1 DESC
```

**Embed em docs**:
```markdown
![Volume Chart](https://dune.com/embeds/123456/789012/chart.png)
```

### Custom Dashboard (React)

```typescript
// Dashboard.tsx
import { useContract } from 'wagmi';
import { useEffect, useState } from 'react';

export default function Dashboard() {
  const [metrics, setMetrics] = useState({
    tvl: '0',
    volume24h: '0',
    users: 0
  });

  const contract = useContract({
    address: CONTRACT_ADDRESS,
    abi: ABI
  });

  useEffect(() => {
    async function fetchMetrics() {
      const tvl = await contract.read.getTotalValueLocked();
      // ... fetch other metrics

      setMetrics({
        tvl: ethers.formatEther(tvl),
        volume24h: '...',
        users: ...
      });
    }

    fetchMetrics();
    const interval = setInterval(fetchMetrics, 60000); // Update every minute

    return () => clearInterval(interval);
  }, [contract]);

  return (
    <div>
      <h1>Protocol Dashboard</h1>

      <div className="metrics">
        <div className="metric">
          <h2>TVL</h2>
          <p>${metrics.tvl}</p>
        </div>

        <div className="metric">
          <h2>24h Volume</h2>
          <p>${metrics.volume24h}</p>
        </div>

        <div className="metric">
          <h2>Users</h2>
          <p>{metrics.users}</p>
        </div>
      </div>

      {/* Charts, tables, etc. */}
    </div>
  );
}
```

---

## 19.6 Incident Response

### Incident Response Plan

**Roles**:
- **Incident Commander**: CEO/CTO - toma decisões finais
- **Tech Lead**: Analisa exploit, propõe fixes
- **Comms Lead**: Community updates, Twitter
- **Multi-sig Signers**: Prontos para assinar emergency txs

### Runbook: Exploit Detected

**Phase 1: STOP THE BLEEDING (0-5min)**

```
1. PAUSE CONTRACT (se possível)
   - Guardian multi-sig: 2-of-3 para speed
   - Ou Defender Autotask auto-pause

2. ALERT EVERYONE
   - Slack: @channel
   - PagerDuty: trigger
   - Discord: pin message

3. ASSEMBLE WAR ROOM
   - Video call link
   - All hands on deck
```

**Phase 2: ASSESS DAMAGE (5-30min)**

```
4. IDENTIFY EXPLOIT
   - Analyze suspicious tx on Tenderly
   - Identify vulnerability
   - How much can be lost (worst case)?

5. DETERMINE RESPONSE
   Options:
   a) Pause sufficient? (if yes, proceed to comms)
   b) Need emergency withdrawal? (pull all funds to multi-sig)
   c) Need whitehat frontrun? (rescue funds before attacker)
   d) Contact attacker? (negotiate return)
```

**Phase 3: EXECUTE RESPONSE (30min-2h)**

```
6. IF EMERGENCY WITHDRAWAL:
   - Multi-sig signs emergency tx
   - Move all funds to safe address
   - Verify on Etherscan

7. IF WHITEHAT RESCUE:
   - Deploy whitehat contract
   - Frontrun attacker with higher gas
   - (Use Flashbots to avoid mempool)

8. IF NEGOTIATION:
   - On-chain message to attacker address
   - Offer bounty (10-20% of funds)
   - "You are a whitehat. Return funds, keep $X"
```

**Phase 4: COMMUNICATE (parallel com Phase 2-3)**

```
9. FIRST UPDATE (within 30min):
   Twitter/Discord:
   "We are investigating a potential issue with our protocol.
    The contract has been paused as a precaution.
    We will provide updates as we learn more."

10. SECOND UPDATE (within 2h):
    "Issue confirmed. [Brief description].
     [Amount] affected.
     No user action required at this time.
     We are working on a fix."

11. DETAILED POST-MORTEM (within 24-48h):
    - What happened
    - Root cause
    - Amount lost/recovered
    - Fix implemented
    - Prevention measures
```

**Phase 5: RECOVERY (days-weeks)**

```
12. FIX VULNERABILITY
    - Patch code
    - Emergency audit
    - Re-audit

13. DEPLOY FIX
    - If upgradeable: upgrade
    - If not: deploy new contract + migration

14. COMPENSATE USERS (if applicable)
    - Snapshot balances
    - Distribute from treasury/insurance
    - Or pro-rata recovery

15. POST-MORTEM REPORT
    - Full transparency
    - Publish detailed analysis
    - Lessons learned
```

### Example: Real Hack Timeline

**Euler Finance ($197M hack - March 2023)**:

```
13:00 UTC: Exploit transaction
13:05 UTC: Team noticed unusual tx
13:10 UTC: Verified exploit
13:15 UTC: No pause function (too late)
13:20 UTC: Public announcement
14:00 UTC: Negotiating with hacker
...
2 weeks later: Hacker returned funds (negotiation successful)
```

**Lessons**:
- ✅ Quick detection (5 min)
- ❌ No pause function (could have saved $197M)
- ✅ Transparent communication
- ✅ Negotiation worked (rare!)

---

## 19.7 Post-Mortem

### Template

```markdown
# Post-Mortem: [Incident Name]

## Summary
- **Date**: 2024-11-14
- **Duration**: 2 hours
- **Impact**: $X lost, Y users affected
- **Status**: Resolved / Funds recovered / Partially recovered

## Timeline (UTC)
- 10:00 - Normal operations
- 10:15 - Suspicious transaction detected
- 10:20 - Contract paused
- 10:30 - Root cause identified
- 11:00 - Fix deployed
- 12:00 - Contract unpaused
- 12:30 - Post-mortem published

## Root Cause
[Technical explanation of vulnerability]

```solidity
// Vulnerable code
function withdraw() external {
    uint amount = balances[msg.sender];
    msg.sender.call{value: amount}(""); // Reentrancy!
    balances[msg.sender] = 0;
}
```

## Impact
- **Direct Loss**: $100,000
- **Users Affected**: 50
- **Recovered**: $90,000 (90%)

## Response
**What went well**:
- ✅ Fast detection (5 minutes)
- ✅ Pause worked correctly
- ✅ Communication was clear

**What went wrong**:
- ❌ Audit missed this case
- ❌ No reentrancy guard
- ❌ Monitoring alert too slow

## Fix
```solidity
// Fixed code
function withdraw() external nonReentrant {
    uint amount = balances[msg.sender];
    balances[msg.sender] = 0; // Update first!
    (bool success,) = msg.sender.call{value: amount}("");
    require(success);
}
```

## Prevention
**Immediate**:
- Added ReentrancyGuard
- Re-audited contract
- Increased monitoring sensitivity

**Long-term**:
- Implement formal verification
- Increase audit budget
- Add circuit breakers

## Compensation
Users will be compensated as follows:
- 90% from recovered funds
- 10% from treasury

## Lessons Learned
1. Always use ReentrancyGuard
2. Pause function is CRITICAL
3. Monitoring saves millions
4. Transparency builds trust
```

---

## 19.8 Communication Strategy

### During Incident

**DO**:
- ✅ Acknowledge quickly (within 30min)
- ✅ Be transparent
- ✅ Provide regular updates (every 1-2h)
- ✅ State facts, not speculation
- ✅ Explain user impact clearly

**DON'T**:
- ❌ Silence (radio silence = panic)
- ❌ Minimize issue ("small bug")
- ❌ Blame others (auditors, users)
- ❌ Promise timeline you can't meet
- ❌ Delete critical tweets

### Template Messages

**First Alert**:
```
We are investigating a potential issue with [Protocol].
The contract has been paused as a precaution.
Your funds are safe. No user action needed.
Updates to follow shortly.
```

**Confirmation**:
```
Issue confirmed. A vulnerability in [Component] has been exploited.
Impact: [Amount] affected.
Status: Contract paused, fix in progress.
No additional user funds at risk.
Detailed update in 2 hours.
```

**Resolution**:
```
The issue has been resolved.
- Vulnerability: [Brief description]
- Impact: [Amount]
- Recovery: [%] of funds recovered
- Fix: Deployed and audited
- Compensation: [Plan]

Full post-mortem: [Link]
Thank you for your patience.
```

---

## 📖 Glossário

**Monitoring**
> Observação contínua de smart contracts para detectar issues.

**Sentinel**
> Watcher que alerta quando condições são met.
> **Ferramenta**: OpenZeppelin Defender.

**Autotask**
> Ação automatizada (ex: auto-pause).

**Post-Mortem**
> Análise pública após incident.
> **Objetivo**: Transparência + aprendizado.

**P0/P1/P2**
> Severity levels de incidents.
> **P0**: Critical (pause tudo AGORA).

**War Room**
> Call de emergência com todos stakeholders.

---

## 🔒 Monitoring Checklist

**Setup** (antes de mainnet):
- [ ] Tenderly configured
- [ ] Defender Sentinels active
- [ ] Discord webhook setup
- [ ] Telegram bot setup
- [ ] PagerDuty integration (P0)
- [ ] Dashboard deployed
- [ ] Incident response plan documented

**Alerts** (configured):
- [ ] Admin function calls
- [ ] Large withdrawals (>$100k)
- [ ] Failed transactions (abnormal rate)
- [ ] Oracle staleness (>1h)
- [ ] Low reserves warning
- [ ] Gas price spikes

**Team** (ready):
- [ ] On-call rotation scheduled
- [ ] War room link shared
- [ ] Multi-sig signers available 24/7
- [ ] Comms lead assigned

**Testing**:
- [ ] Test alert delivery
- [ ] Test auto-pause mechanism
- [ ] Run incident simulation
- [ ] Verify dashboard accuracy

---

## 📚 Recursos

### Tools
- **[Tenderly](https://tenderly.co)**
- **[OpenZeppelin Defender](https://defender.openzeppelin.com)**
- **[Dune Analytics](https://dune.com)**
- **[Forta](https://forta.org)** - Threat detection

### Post-Mortems (Learn)
- **[Rekt News](https://rekt.news)** - Hack análises
- **[BlockSec](https://blocksec.com)** - Incident reports

---

## 🎯 Conclusão

**Monitoring = Insurance.**

Invista em monitoring:
- Detecta exploits cedo (minutos vs horas)
- Auto-pause salva fundos
- Transparência mantém confiança

**Never skimp on monitoring.**

---

**FIM DO EBOOK PRINCIPAL!** 🎉

Próximo: Apêndices (Comparativo, Glossário, Recursos)

---

**Autor**: Baseado em incidents reais + best practices
**Última Atualização**: 2025-11-14
