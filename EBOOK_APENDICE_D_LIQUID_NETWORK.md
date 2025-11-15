# Apêndice D: Liquid Network - Sidechain Bitcoin para Desenvolvedores

> **Para Desenvolvedores Experientes**: Se você conhece Bitcoin e busca uma plataforma que combine a segurança do Bitcoin com privacidade avançada, confirmações rápidas e emissão de ativos, Liquid Network é a sidechain federada que oferece isso através de Confidential Transactions e uma arquitetura única baseada em Strong Federations.

---

## Índice

- [D.1 Introdução ao Liquid Network](#d1-introdução-ao-liquid-network)
- [D.2 Arquitetura e Modelo de Consenso](#d2-arquitetura-e-modelo-de-consenso)
- [D.3 Confidential Transactions - Privacidade por Padrão](#d3-confidential-transactions---privacidade-por-padrão)
- [D.4 LBTC - Liquid Bitcoin e o Mecanismo de Peg](#d4-lbtc---liquid-bitcoin-e-o-mecanismo-de-peg)
- [D.5 Issued Assets - Tokens Nativos no Liquid](#d5-issued-assets---tokens-nativos-no-liquid)
- [D.6 USDT e Stablecoins no Liquid](#d6-usdt-e-stablecoins-no-liquid)
- [D.7 Desenvolvimento no Liquid - Setup e Ambiente](#d7-desenvolvimento-no-liquid---setup-e-ambiente)
- [D.8 SDKs e APIs para Desenvolvimento](#d8-sdks-e-apis-para-desenvolvimento)
- [D.9 Simplicity - Smart Contracts no Liquid](#d9-simplicity---smart-contracts-no-liquid)
- [D.10 Desenvolvendo Wallets para Liquid](#d10-desenvolvendo-wallets-para-liquid)
- [D.11 Emissão de Assets - Do Código ao Registry](#d11-emissão-de-assets---do-código-ao-registry)
- [D.12 Comparativo - Liquid vs Lightning vs Rootstock](#d12-comparativo---liquid-vs-lightning-vs-rootstock)
- [D.13 Casos de Uso e Ecossistema](#d13-casos-de-uso-e-ecossistema)
- [D.14 Segurança e Trade-offs](#d14-segurança-e-trade-offs)
- [📖 Glossário Consolidado](#-glossário-consolidado)
- [📚 Recursos e Documentação](#-recursos-e-documentação)
- [🎯 Próximos Passos](#-próximos-passos)

---

## D.1 Introdução ao Liquid Network

### O Que É Liquid Network

**Liquid Network** é uma sidechain do Bitcoin construída usando a plataforma open-source **Elements**, que permite:

- **Confirmações rápidas**: Blocos de ~1 minuto (vs 10 min do Bitcoin)
- **Privacidade por padrão**: Confidential Transactions ocultam valores e tipos de asset
- **Emissão de ativos**: Tokens nativos sem necessidade de smart contracts complexos
- **Peg 1:1 com Bitcoin**: LBTC (Liquid Bitcoin) mantém paridade com BTC

### 📖 Glossário de Termos

**Sidechain**
> Blockchain separada que roda em paralelo ao Bitcoin, conectada através de um mecanismo de peg bidirecional.
>
> **Analogia Web2**: Como um microserviço que se comunica com o serviço principal através de APIs específicas.
>
> **Por que existe**: Permite experimentação e recursos avançados sem modificar o Bitcoin.

**Elements**
> Plataforma open-source de blockchain desenvolvida pela Blockstream, base do Liquid Network.
>
> **Analogia**: Como um fork do Bitcoin Core com features adicionais (Confidential Transactions, Issued Assets, etc.).

**LBTC (Liquid Bitcoin)**
> Representação 1:1 do Bitcoin dentro da Liquid Network.
>
> **Importante**: 1 LBTC = 1 BTC (sempre). É o Bitcoin "preso" na sidechain através do mecanismo de peg.

### Por Que Liquid Existe

**Problemas que resolve**:

1. **Velocidade**: Bitcoin mainnet é lento (10 min/bloco)
2. **Privacidade**: Bitcoin é pseudônimo, não anônimo (todos valores são públicos)
3. **Custos**: Fees do Bitcoin podem ser altos em períodos de congestionamento
4. **Emissão de ativos**: Bitcoin não suporta nativamente tokens (apenas via Taproot Assets, Omni, etc.)

**Trade-offs**:

| Aspecto | Bitcoin Mainnet | Liquid Network |
|---------|----------------|----------------|
| **Descentralização** | 🟢 Máxima (milhares de mineradores) | 🟡 Federada (15 functionaries) |
| **Segurança** | 🟢 PoW global | 🟡 Multisig 11-of-15 |
| **Velocidade** | 🔴 ~10 min/bloco | 🟢 ~1 min/bloco |
| **Privacidade** | 🔴 Valores públicos | 🟢 Confidential Transactions |
| **Finalidade** | 🟡 Probabilística (6 confirms) | 🟢 Determinística (assinaturas) |
| **Custos** | 🟡 Variável (alto em congestão) | 🟢 Baixo e previsível |

### Ecossistema Atual (2024)

- **TVL**: ~$500M em assets na rede
- **Functionaries**: 15 entidades (exchanges, empresas blockchain)
- **Assets emitidos**: 100+ (USDT, EURX, stablecoins, securities)
- **Simplicity**: Live em mainnet desde julho 2024

---

## D.2 Arquitetura e Modelo de Consenso

### Strong Federations - Consenso Federado

Liquid não usa Proof of Work nem Proof of Stake. Usa **Strong Federations**:

**Conceito**:
- Grupo de entidades mutuamente desconfiadas (**functionaries**)
- Cada functionary opera **Hardware Security Modules (HSMs)** customizados
- Consenso requer **⅔ dos functionaries** (11 de 15)

#### Functionaries e Seus Papéis Duais

Cada functionary desempenha **dois papéis**:

**1. Blocksigner** (produção de blocos):
```
┌──────────────────────────────────────────────────┐
│           Blocksigner Operation                  │
├──────────────────────────────────────────────────┤
│                                                  │
│  1. Functionary propõe bloco (round-robin)      │
│         ↓                                        │
│  2. Distribui para outros functionaries         │
│         ↓                                        │
│  3. Cada um valida e assina                     │
│         ↓                                        │
│  4. Combina assinaturas (≥11 de 15)            │
│         ↓                                        │
│  5. Adiciona à blockchain Liquid                │
│                                                  │
│  ⏱️ Tempo: ~1 minuto por bloco                   │
└──────────────────────────────────────────────────┘
```

**2. Watchmen** (guarda dos bitcoins):
```
┌──────────────────────────────────────────────────┐
│            Watchmen Operation                    │
├──────────────────────────────────────────────────┤
│                                                  │
│  • Gerenciam multisig 11-of-15                  │
│  • Seguram BTC que está "pegado" no Liquid      │
│  • Processam peg-ins (BTC → LBTC)              │
│  • Processam peg-outs (LBTC → BTC)             │
│                                                  │
│  🔒 BTC guardado: ~3.500 BTC (~$350M)           │
└──────────────────────────────────────────────────┘
```

### Hardware Security Modules (HSMs)

**Por que HSMs?**

- Impedem que functionaries extraiam chaves privadas
- Chaves nunca deixam o hardware
- Software customizado roda dentro do HSM
- Geograficamente e geopoliticamente distribuídos

**Lista de Functionaries** (alguns exemplos):
1. Bitfinex
2. BTSE
3. Bullish
4. Coinone
5. CryptoGarage
6. DG Lab
7. Gameshift (Solana)
8. GOPAX
9. Jan3
10. Sideswap
11. Xapo Bank
12. ... (15 total)

### Produção de Blocos

**Round-Robin Proposal**:

```solidity
// Pseudo-código do processo
contract LiquidBlockProduction {
    address[15] public functionaries;
    uint8 public constant THRESHOLD = 11; // 2/3 + 1

    function proposeBlock(uint256 round) internal {
        // Functionary da vez propõe
        address proposer = functionaries[round % 15];

        // Cria bloco não-assinado
        Block memory newBlock = createUnsignedBlock();

        // Distribui para assinatura
        distributeForSigning(newBlock);
    }

    function signBlock(Block memory block) internal {
        // Cada functionary valida
        require(validateBlock(block), "Invalid block");

        // Assina dentro do HSM
        Signature memory sig = hsmSign(block);

        // Envia assinatura
        broadcastSignature(sig);
    }

    function finalizeBlock(Block memory block, Signature[] memory sigs) internal {
        // Combina assinaturas
        require(sigs.length >= THRESHOLD, "Not enough signatures");

        // Adiciona à chain
        addToBlockchain(block, combineSignatures(sigs));
    }
}
```

### Código Real - Interação com elementsd

**Verificar functionaries**:

```bash
# Via elements-cli
elements-cli getblockchaininfo

# Exemplo de output
{
  "chain": "liquidv1",
  "blocks": 3245678,
  "headers": 3245678,
  "bestblockhash": "abc123...",
  "difficulty": 1,
  "mediantime": 1732345678,
  "verificationprogress": 0.99999,
  "initialblockdownload": false,
  "chainwork": "...",
  "size_on_disk": 45123456789,
  "pruned": false,
  "signblock_asm": "OP_5 <pubkey1> <pubkey2> ... <pubkey15> OP_15 OP_CHECKMULTISIG",
  "max_block_witness": 3000000
}
```

---

## D.3 Confidential Transactions - Privacidade por Padrão

### O Que São Confidential Transactions

**Problema no Bitcoin**:
```
Transação Bitcoin:
  From: 1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
  To:   1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2
  Value: 2.5 BTC  ← PÚBLICO (todos veem)
  Asset: BTC      ← PÚBLICO (todos veem)
```

**Solução no Liquid**:
```
Transação Liquid (Confidential):
  From: VJLAx9... (confidential address)
  To:   VTpvv8... (confidential address)
  Value: ??? (OCULTADO via blinding)
  Asset: ??? (OCULTADO via blinding)

  ✅ Apenas sender e receiver conhecem valores
  ✅ Observadores veem apenas: "alguma quantia de algum asset foi transferida"
```

### Tecnologia: Pedersen Commitments

**Matemática por trás** (simplificado):

```python
# Commitment = valor_real * G + blinding_factor * H
# Onde G e H são pontos da curva elíptica

class PedersenCommitment:
    def __init__(self, value, blinding_factor):
        self.value = value
        self.blinding = blinding_factor

    def commit(self):
        """
        Cria commitment que oculta o valor
        Mas permite verificação matemática
        """
        # C = vG + bH
        commitment = (self.value * G) + (self.blinding * H)
        return commitment

    def verify_sum(commitments):
        """
        Verifica que inputs = outputs
        Sem revelar valores individuais
        """
        # Soma dos commitments de input = soma dos commitments de output
        # Se verdadeiro, transação é válida (sem inflation)
        sum_inputs = sum(c_in for c_in in input_commitments)
        sum_outputs = sum(c_out for c_out in output_commitments)

        return sum_inputs == sum_outputs
```

**Range Proofs**:

Problema: Como provar que um valor oculto não é negativo?

```python
class RangeProof:
    """
    Prova que o valor commitado está em um range
    Ex: 0 <= valor <= 2^64
    Sem revelar o valor exato
    """
    def generate(self, value, blinding_factor):
        # Bulletproofs (compact range proofs)
        # Prova matemática de que:
        # 0 ≤ value < 2^64

        proof = bulletproof_generate(value, blinding_factor)
        return proof

    def verify(self, commitment, proof):
        # Verifica proof sem conhecer valor
        return bulletproof_verify(commitment, proof)
```

### Confidential Addresses

**Estrutura**:

```javascript
// Address normal (Bitcoin)
const btcAddress = "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa";

// Confidential Address (Liquid)
const liquidAddress = "VJLAx9k3qQ8h5LsJx3DfT9mHW7K9vN5aBc2Pp1qRs...";
//                      ↑ Prefix (V = mainnet, t = testnet)
//                        ↑ Base address
//                          ↑ Public blinding key

// Decompondo
interface ConfidentialAddress {
  prefix: "V" | "t";
  baseAddress: string;      // Endereço normal
  blindingPubKey: string;   // Chave pública para blinding
}
```

**Criando Confidential Address**:

```bash
# Via elements-cli
elements-cli getnewaddress

# Output
VJLAx9k3qQ8h5LsJx3DfT9mHW7K9vN5aBc2Pp1qRs8mQvK9...

# Obter detalhes
elements-cli getaddressinfo VJLAx9k3qQ8h5LsJx3DfT9mHW7K9vN5aBc2Pp1qRs8mQvK9...

{
  "address": "VJLAx9k3qQ8h5LsJx3DfT9mHW7K9vN5aBc2Pp1qRs8mQvK9...",
  "scriptPubKey": "0014abc123...",
  "ismine": true,
  "iswatchonly": false,
  "isscript": false,
  "iswitness": true,
  "witness_version": 0,
  "witness_program": "abc123...",
  "confidential_key": "02def456...",  // ← Blinding pubkey
  "unconfidential": "ex1qxyz789..."   // ← Base address
}
```

### Unblinding - Revelando Dados

**Quando compartilhar dados**:

```javascript
// Cenário: Compliance, auditoria, etc.
class TransactionUnblinding {
  async shareUnblindingData(txid, recipient) {
    // 1. Obter blinding key da transação
    const tx = await elements.getRawTransaction(txid, true);

    // 2. Exportar dados de unblinding
    const unblindingData = {
      txid: txid,
      vout: 0,
      asset: tx.vout[0].asset,           // Revelado
      assetamount: tx.vout[0].value,     // Revelado
      assetblinder: "abc123...",         // Blinding factor do asset
      amountblinder: "def456..."         // Blinding factor do valor
    };

    // 3. Compartilhar com terceiro (ex: auditor)
    return unblindingData;
  }

  async verifyWithUnblindingData(unblindingData) {
    // Terceiro pode verificar commitment
    const tx = await elements.getRawTransaction(unblindingData.txid, true);

    // Recriar commitment com dados revelados
    const recreatedCommitment = createCommitment(
      unblindingData.assetamount,
      unblindingData.amountblinder
    );

    // Verificar que bate com commitment na blockchain
    return recreatedCommitment === tx.vout[0].commitment;
  }
}
```

**Comando elements-cli**:

```bash
# Exportar unblinding data
elements-cli dumpblindingkey <confidential_address>

# Output: chave privada de blinding (CUIDADO: sensível!)
cVt4o7BGAig1UXywgGSmARhxMdzP5qvQsxKkSsc1XEkw3tDTQFpy

# Importar para auditor
elements-cli importblindingkey <confidential_address> <blinding_key>
```

### Tamanho das Transações

**Trade-off: Privacidade vs Tamanho**

```
Bitcoin Transaction:    ~250 bytes
Liquid Transaction:     ~5.000 bytes (20x maior!)

Por quê?
- Range Proofs: ~2.500 bytes por output
- Commitments extras
- Assinaturas adicionais
```

**Otimização - Bulletproofs**:

Liquid usa Bulletproofs para reduzir range proofs:

```
Range Proof clássico:  ~3.000 bytes/output
Bulletproof:          ~700 bytes/output

Economia: ~77%
```

---

## D.4 LBTC - Liquid Bitcoin e o Mecanismo de Peg

### Peg Bidirecional (Two-Way Peg)

**Conceito**:

```
Bitcoin Mainnet  ⟷  Liquid Network
      BTC         ⟷      LBTC

Sempre: 1 BTC = 1 LBTC (paridade mantida)
```

### Peg-In: BTC → LBTC

**Processo completo**:

```
┌─────────────────────────────────────────────────────┐
│           PEG-IN PROCESS (BTC → LBTC)              │
├─────────────────────────────────────────────────────┤
│                                                     │
│ 1. Usuário: Gera endereço de peg-in no Liquid     │
│    elements-cli getpeginaddress                    │
│         ↓                                          │
│ 2. Retorna: {                                      │
│      "mainchain_address": "3...",  ← Envia BTC aqui│
│      "claim_script": "0014abc..."  ← Guarda isso!  │
│    }                                               │
│         ↓                                          │
│ 3. Usuário: Envia BTC para mainchain_address      │
│    bitcoin-cli sendtoaddress 3... 1.5              │
│         ↓                                          │
│ 4. Aguarda: 102 confirmações no Bitcoin           │
│    ⏱️ Tempo: ~17 horas                             │
│         ↓                                          │
│ 5. Usuário: Reivindica LBTC no Liquid            │
│    elements-cli claimpegin <txid> <proof>          │
│         ↓                                          │
│ 6. LBTC creditado na carteira Liquid              │
│                                                     │
│ ✅ Total: ~17-18 horas                             │
└─────────────────────────────────────────────────────┘
```

**Código real - Peg-in completo**:

```bash
#!/bin/bash

# ========================================
# PEG-IN: BTC → LBTC
# ========================================

# Passo 1: Gerar endereço de peg-in no Liquid
echo "Gerando endereço de peg-in..."
PEG_ADDRESS=$(elements-cli getpeginaddress)

MAINCHAIN_ADDR=$(echo $PEG_ADDRESS | jq -r '.mainchain_address')
CLAIM_SCRIPT=$(echo $PEG_ADDRESS | jq -r '.claim_script')

echo "Envie BTC para: $MAINCHAIN_ADDR"
echo "Guarde claim_script: $CLAIM_SCRIPT"

# Passo 2: Enviar BTC (do Bitcoin Core)
echo "Enviando 1.5 BTC..."
BTC_TXID=$(bitcoin-cli sendtoaddress $MAINCHAIN_ADDR 1.5)

echo "TX Bitcoin: $BTC_TXID"
echo "Aguardando 102 confirmações (~17 horas)..."

# Aguardar confirmações
while [ $(bitcoin-cli gettransaction $BTC_TXID | jq '.confirmations') -lt 102 ]; do
  CONFS=$(bitcoin-cli gettransaction $BTC_TXID | jq '.confirmations')
  echo "Confirmações: $CONFS/102"
  sleep 600  # Check a cada 10 minutos
done

echo "✅ 102 confirmações atingidas!"

# Passo 3: Obter proof da transação Bitcoin
BTC_TX_HEX=$(bitcoin-cli getrawtransaction $BTC_TXID)
BTC_PROOF=$(bitcoin-cli gettxoutproof "[\"$BTC_TXID\"]")

# Passo 4: Reivindicar LBTC no Liquid
echo "Reivindicando LBTC..."
CLAIM_TXID=$(elements-cli claimpegin "$BTC_TX_HEX" "$BTC_PROOF" "$CLAIM_SCRIPT")

echo "✅ Peg-in completo!"
echo "LBTC claim TX: $CLAIM_TXID"

# Verificar saldo
elements-cli getbalance
```

**Por que 102 confirmações?**

- Evitar reorganizações profundas no Bitcoin
- Segurança contra double-spends
- Trade-off: segurança vs velocidade

⚠️ **Warning**: Endereços de peg-in **não são reutilizáveis**. Gere novo para cada peg-in.

### Peg-Out: LBTC → BTC

**Processo completo**:

```
┌─────────────────────────────────────────────────────┐
│           PEG-OUT PROCESS (LBTC → BTC)             │
├─────────────────────────────────────────────────────┤
│                                                     │
│ 1. Usuário: Inicia peg-out no Liquid              │
│    elements-cli sendtomainchain <btc_addr> <amt>   │
│         ↓                                          │
│ 2. TX vai para mempool do Liquid                  │
│         ↓                                          │
│ 3. Watchmen detectam requisição                   │
│    (monitoram constantemente)                      │
│         ↓                                          │
│ 4. Watchmen processam em batches                  │
│    • Round a cada ~17 minutos                     │
│    • Agrupam múltiplos peg-outs                   │
│         ↓                                          │
│ 5. Criam transação Bitcoin multisig 11-of-15      │
│         ↓                                          │
│ 6. Assinam e broadcast no Bitcoin                 │
│         ↓                                          │
│ 7. BTC chega após confirmações Bitcoin            │
│    ⏱️ Tempo: ~10-60 minutos (depende do batch)     │
│         ↓                                          │
│ 8. ✅ BTC creditado no endereço Bitcoin           │
│                                                     │
│ Total: ~30-90 minutos típico                       │
└─────────────────────────────────────────────────────┘
```

**Código real - Peg-out**:

```bash
#!/bin/bash

# ========================================
# PEG-OUT: LBTC → BTC
# ========================================

# Verificar saldo LBTC
echo "Saldo LBTC atual:"
elements-cli getbalance

# Definir destino Bitcoin
BTC_DESTINATION="1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"
AMOUNT="0.5"  # LBTC

# Iniciar peg-out
echo "Iniciando peg-out de $AMOUNT LBTC..."
PEGOUT_TXID=$(elements-cli sendtomainchain "$BTC_DESTINATION" $AMOUNT)

echo "Peg-out TX: $PEGOUT_TXID"
echo "Status: Aguardando processamento pelos Watchmen..."

# Monitorar status
elements-cli gettransaction $PEGOUT_TXID

# Output example:
# {
#   "amount": -0.50000000,
#   "fee": -0.00001000,
#   "confirmations": 3,
#   "blockhash": "abc123...",
#   "blocktime": 1732345678,
#   ...
# }

echo "Aguardando próximo batch dos Watchmen (~17 min)..."
echo "BTC será enviado para $BTC_DESTINATION"
```

**Limitação importante**:

⚠️ **Apenas Federation Members podem iniciar peg-outs**. Usuários comuns precisam:
- Usar exchange que seja membro (ex: Bitfinex, BTSE)
- Ou: Fazer swap LBTC → BTC via DEX/P2P

**Peg-out via Exchange** (método mais comum):

```javascript
// Exemplo conceitual - Bitfinex API
class LiquidPegOut {
  async withdrawLBTCtoBTC(amount) {
    // 1. Usuário tem LBTC na exchange
    // 2. Solicita saque para Bitcoin
    const withdrawal = await bitfinexAPI.withdraw({
      currency: 'LBTC',
      method: 'bitcoin',  // Destino
      amount: amount,
      address: 'btc_address_here'
    });

    // 3. Exchange processa internamente
    //    - Queima LBTC do usuário
    //    - Inicia peg-out como federation member
    //    - Envia BTC após confirmações

    return withdrawal;
  }
}
```

---

## D.5 Issued Assets - Tokens Nativos no Liquid

### O Que São Issued Assets

**Conceito**:

Liquid permite criar **tokens nativos** sem smart contracts:

```
Bitcoin:     Apenas BTC
Ethereum:    ETH + tokens ERC-20 (via smart contracts)
Liquid:      LBTC + Issued Assets (nativos, sem código)
```

**Vantagens**:

✅ **Mesmo padrão para todos**: LBTC e tokens usam mesma estrutura
✅ **Atomic swaps nativos**: Troca entre assets sem intermediário
✅ **Multisig funciona igual**: Mesmo modelo de segurança
✅ **Confidential por padrão**: Valores e tipos ocultos
✅ **Sem bugs de smart contract**: Não há código customizado vulnerável

### Anatomia de um Issued Asset

**Estrutura**:

```javascript
interface IssuedAsset {
  // Identificador único (64 hex chars)
  assetId: "6f0279e9ed041c3d710a9f57d0c02928416460c4b722ae3457a11eec381c526d";

  // Metadata (opcional, no registry)
  name: "Tether USD";
  ticker: "USDt";
  precision: 8;  // Decimais
  domain: "tether.to";

  // Emissão
  totalIssued: 1000000000;  // Total emitido
  canReissue: true;         // Pode emitir mais?

  // Contract (opcional - hash de contrato legal)
  contractHash: "abc123...";
}
```

**Exemplo real - USDT no Liquid**:

```bash
# Asset ID do USDT
USDT_ASSET_ID="ce091c998b83c78bb71a632313ba3760f1763d9cfcffae02258ffa9865a37bd2"

# Consultar asset
elements-cli getassetdata $USDT_ASSET_ID

# Output
{
  "asset": "ce091c998b83...",
  "name": "Tether USD",
  "ticker": "USDt",
  "precision": 8,
  "entity": {
    "domain": "tether.to"
  },
  "version": 0
}
```

### Emitindo um Asset - Código Completo

**Passo a passo**:

```bash
#!/bin/bash

# ========================================
# ISSUED ASSET CREATION
# ========================================

# Passo 1: Criar reissuance token (para emitir mais no futuro)
echo "Criando asset..."

# Quantidade inicial
INITIAL_AMOUNT=1000000

# Emitir asset
ISSUANCE=$(elements-cli issueasset $INITIAL_AMOUNT 1 false)

# Parse do resultado
ASSET_ID=$(echo $ISSUANCE | jq -r '.asset')
TOKEN_ID=$(echo $ISSUANCE | jq -r '.token')
ENTROPY=$(echo $ISSUANCE | jq -r '.entropy')
TX_ID=$(echo $ISSUANCE | jq -r '.txid')
VIN=$(echo $ISSUANCE | jq -r '.vin')

echo "✅ Asset emitido!"
echo "Asset ID: $ASSET_ID"
echo "Token ID (reissuance): $TOKEN_ID"
echo "TX: $TX_ID"

# Passo 2: Verificar saldo do novo asset
elements-cli getbalance "*" 1 false "$ASSET_ID"
# Output: 1000000

# Passo 3: Re-emitir mais tokens (se tiver reissuance token)
echo "Re-emitindo 500.000 tokens..."
REISSUANCE=$(elements-cli reissueasset "$ASSET_ID" 500000)

echo "Total agora: 1.500.000 tokens"
```

**Com metadata (registry)**:

```javascript
// Via Blockstream AMP API ou manual
const assetMetadata = {
  asset_id: "abc123...",

  // Identificação
  name: "My Stablecoin",
  ticker: "MSTB",
  precision: 8,

  // Entidade emissora
  entity: {
    domain: "mystablecoin.com"  // Deve provar controle do domínio
  },

  // Contrato (hash do PDF, etc.)
  contract: {
    entity: {
      domain: "mystablecoin.com"
    },
    issuer_pubkey: "02abc...",
    name: "My Stablecoin Terms",
    ticker: "MSTB",
    version: 0
  }
};

// Submeter para registry
await submitToLiquidAssetRegistry(assetMetadata);
```

**Verificação de domínio** (prova de controle):

```json
// Arquivo .well-known/liquid-asset-proof-[asset_id]
// Hospedado em https://mystablecoin.com/.well-known/...

{
  "asset_id": "abc123...",
  "contract": {
    "entity": {
      "domain": "mystablecoin.com"
    },
    "issuer_pubkey": "02def...",
    "name": "My Stablecoin",
    "ticker": "MSTB",
    "version": 0
  }
}
```

### Transferindo Issued Assets

**Igual a LBTC** (mesma API):

```bash
# Enviar 100 tokens do asset personalizado
RECIPIENT="VJLAx9k3qQ8h5LsJx3DfT9mHW7K9vN5aBc2Pp1qRs..."

elements-cli sendtoaddress \
  "$RECIPIENT" \
  100 \
  "" \
  "" \
  false \
  false \
  1 \
  "UNSET" \
  false \
  "$ASSET_ID"  # ← Especifica o asset

# Confidential por padrão!
# Observadores não veem:
# - Quanto foi enviado
# - Qual asset foi enviado
```

### Atomic Swaps Entre Assets

**Troca sem intermediário**:

```javascript
class LiquidAtomicSwap {
  /**
   * Alice: Quer trocar 100 USDT por 0.005 LBTC
   * Bob: Quer o inverso
   * Troca atômica (tudo ou nada)
   */
  async createAtomicSwap(alice, bob) {
    // Alice cria proposta
    const aliceProposal = await elements.createrawtransaction(
      // Inputs de Alice
      [{ txid: alice.utxo.txid, vout: alice.utxo.vout }],
      // Outputs
      {
        [bob.address]: { [USDT_ASSET]: 100 }  // Alice envia USDT
      }
    );

    // Alice assina parcialmente
    const alicePartialSign = await elements.signrawtransactionwithwallet(
      aliceProposal,
      [],
      "ALL"
    );

    // Bob adiciona seu input e output
    const completeTx = await elements.createrawtransaction(
      // Inputs: Alice + Bob
      [
        { txid: alice.utxo.txid, vout: alice.utxo.vout },
        { txid: bob.utxo.txid, vout: bob.utxo.vout }
      ],
      // Outputs: troca
      {
        [bob.address]: { [USDT_ASSET]: 100 },    // Bob recebe USDT
        [alice.address]: { [LBTC_ASSET]: 0.005 } // Alice recebe LBTC
      }
    );

    // Bob assina
    const bobSign = await elements.signrawtransactionwithwallet(completeTx);

    // Alice assina final
    const finalSign = await elements.signrawtransactionwithwallet(bobSign.hex);

    // Broadcast
    if (finalSign.complete) {
      const txid = await elements.sendrawtransaction(finalSign.hex);
      return { success: true, txid };
    }

    // Se falhar, nada acontece (atômico)
    return { success: false };
  }
}
```

---

## D.6 USDT e Stablecoins no Liquid

### Tether (USDt) no Liquid

**Lançamento**: Julho de 2019 (um dos primeiros assets)

**Asset ID**:
```
ce091c998b83c78bb71a632313ba3760f1763d9cfcffae02258ffa9865a37bd2
```

**Por que USDT escolheu Liquid?**

1. **Privacidade**: Empresas querem confidencialidade em transações
2. **Velocidade**: Confirmações em 2 minutos vs 10-60 min (Bitcoin/Ethereum)
3. **Custos**: Fees baixos e previsíveis
4. **Compliance**: Pode compartilhar unblinding data com auditores

### Usando USDT no Liquid

**Obter saldo**:

```bash
# Asset ID do USDT
USDT="ce091c998b83c78bb71a632313ba3760f1763d9cfcffae02258ffa9865a37bd2"

# Saldo em USDT
elements-cli getbalance "*" 1 false "$USDT"

# Listar todos assets na wallet
elements-cli getwalletinfo

# Output:
# {
#   "balance": {
#     "bitcoin": 1.5,  // LBTC
#     "ce091c99...": 1000.0  // USDT
#   }
# }
```

**Enviar USDT**:

```bash
RECIPIENT="VJLAx9k3qQ8h5LsJx3DfT9mHW7K9vN5aBc..."
AMOUNT=100

elements-cli sendtoaddress \
  "$RECIPIENT" \
  $AMOUNT \
  "" \
  "" \
  false \
  false \
  1 \
  "UNSET" \
  false \
  "$USDT"

# Transação confidential:
# - Valor oculto
# - Asset oculto (observador não sabe se é USDT, LBTC ou outro)
```

### Outras Stablecoins no Liquid

**EURx (Euro)** - Stablecoin EUR:
```
Asset ID: 18729918ab4bca843656f08d4dd877bed6641fbd596a0a963abbf199cfeb3cec
Emissor: Quantoz
```

**LCAD (Canadian Dollar)**:
```
Liquid Canadian Dollar
Emissor: Bull Bitcoin
```

**Vantagens para emissores**:

| Aspecto | Ethereum | Liquid |
|---------|----------|--------|
| **Privacidade** | 🔴 Pública | 🟢 Confidencial |
| **Velocidade** | 🟡 15-60s | 🟢 ~2 min (final) |
| **Custos** | 🔴 Alto (gas variável) | 🟢 Baixo (fixo) |
| **Atomic swaps** | 🟡 Via DEX | 🟢 Nativo |
| **Compliance** | 🔴 Difícil (público) | 🟢 Unblinding seletivo |
| **Bugs de contrato** | 🔴 Possível | 🟢 N/A (nativo) |

---

## D.7 Desenvolvimento no Liquid - Setup e Ambiente

### Instalação do Elements (Liquid Node)

**Linux (Ubuntu/Debian)**:

```bash
# Instalar dependências
sudo apt-get update
sudo apt-get install -y \
  build-essential libtool autotools-dev automake pkg-config \
  libssl-dev libevent-dev bsdmainutils libboost-all-dev \
  libdb++-dev libminiupnpc-dev libzmq3-dev

# Clonar repositório Elements
git clone https://github.com/ElementsProject/elements.git
cd elements

# Checkout versão estável (ex: v22.1.1)
git checkout v22.1.1

# Compilar
./autogen.sh
./configure --disable-tests --disable-bench
make -j$(nproc)
sudo make install

# Verificar instalação
elementsd --version
# Output: Elements Core Daemon version v22.1.1
```

**macOS (via Homebrew)**:

```bash
brew install elements
```

**Docker** (método mais fácil):

```bash
# Pull imagem oficial
docker pull blockstream/elementsd:latest

# Rodar node
docker run -d \
  --name liquid-node \
  -v $HOME/.elements:/root/.elements \
  -p 7041:7041 \
  -p 7040:7040 \
  blockstream/elementsd:latest \
  elementsd -chain=liquidv1 -rpcuser=user -rpcpassword=pass
```

### Configuração - elements.conf

**Arquivo**: `~/.elements/elements.conf`

```ini
# ========================================
# LIQUID MAINNET CONFIG
# ========================================

# Network
chain=liquidv1

# RPC
rpcuser=your_username
rpcpassword=your_secure_password
rpcallowip=127.0.0.1
rpcport=7041

# Para desenvolvimento: permitir localhost
rpcbind=127.0.0.1

# Conexão com Bitcoin (para validar peg-ins)
# Opcional: apenas se quiser validar peg-ins localmente
validatepegin=1
mainchainrpchost=127.0.0.1
mainchainrpcport=8332
mainchainrpcuser=bitcoin_user
mainchainrpcpassword=bitcoin_pass

# ZMQ (para escutar eventos)
zmqpubrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28333
zmqpubhashblock=tcp://127.0.0.1:28334

# Indexação (para consultas avançadas)
txindex=1
assetdir=1

# Logging
debug=rpc
```

**Testnet config**:

```ini
# LIQUID TESTNET
chain=liquidtestnet

rpcuser=testuser
rpcpassword=testpass
rpcport=7041

# Testnet não precisa de Bitcoin mainnet
validatepegin=0

txindex=1
```

### Primeiros Comandos

**Iniciar node**:

```bash
# Mainnet
elementsd -daemon

# Testnet
elementsd -chain=liquidtestnet -daemon

# Verificar sincronização
elements-cli getblockchaininfo

# Output:
# {
#   "chain": "liquidv1",
#   "blocks": 3245678,
#   "headers": 3245678,
#   "bestblockhash": "abc123...",
#   "verificationprogress": 0.99999,
#   "initialblockdownload": false,
#   ...
# }
```

**Criar wallet**:

```bash
# Criar nova wallet
elements-cli createwallet "mywallet"

# Listar wallets
elements-cli listwallets

# Obter novo endereço
elements-cli getnewaddress

# Output: VJLAx9k3qQ8h5LsJx3DfT9mHW7K9vN5aBc...

# Verificar saldo
elements-cli getbalance

# Listar UTXOs
elements-cli listunspent
```

### Testnet - Obter Fundos

**Faucets**:

```bash
# 1. Gerar endereço testnet
elements-cli -chain=liquidtestnet getnewaddress

# 2. Usar faucet (URL hipotética)
# https://liquidtestnet.com/faucet
# Cole seu endereço e receba LBTC testnet

# 3. Verificar recebimento
elements-cli -chain=liquidtestnet getbalance
```

---

## D.8 SDKs e APIs para Desenvolvimento

### Green Development Kit (GDK)

**O que é**: SDK multi-plataforma da Blockstream para wallets

**Instalação**:

```bash
# Python
pip install greenaddress

# Download release
wget https://github.com/Blockstream/gdk/releases/download/release_X.Y.Z/gdk-python-wheel-linux-amd64.zip
unzip gdk-python-wheel-linux-amd64.zip
pip install gdk-*.whl
```

**Exemplo - Conectar e obter saldo**:

```python
import greenaddress as gdk
import json

# Inicializar sessão
session = gdk.Session({'name': 'liquidv1'})

# Conectar à rede
session.connect()

# Criar/restaurar wallet
mnemonic = gdk.generate_mnemonic()
credentials = {'mnemonic': mnemonic}

# Login
session.register_user({}, credentials).resolve()
session.login({}, credentials).resolve()

# Obter endereço de recebimento
address_data = session.get_receive_address({'subaccount': 0}).resolve()
print(f"Endereço: {address_data['address']}")

# Obter saldo
balance = session.get_balance({'subaccount': 0, 'num_confs': 0}).resolve()
print(f"LBTC: {balance['bitcoin']}")

# Enviar transação
tx_data = {
    'addressees': [{
        'address': 'VJL...',  # Destinatário
        'satoshi': 100000,    # 0.001 LBTC
        'asset_id': '6f0279e9...'  # LBTC asset ID
    }]
}

tx = session.create_transaction(tx_data).resolve()
signed = session.sign_transaction(tx).resolve()
sent = session.send_transaction(signed).resolve()

print(f"TX: {sent['txhash']}")
```

**Escutar eventos**:

```python
# Callback para notificações
def notification_handler(session, notification):
    event = notification['event']

    if event == 'block':
        print(f"Novo bloco: {notification['block']['block_height']}")

    elif event == 'transaction':
        tx = notification['transaction']
        print(f"Nova TX: {tx['txhash']}")
        print(f"Valor: {tx['satoshi']['bitcoin']}")

# Registrar handler
session.set_notification_handler(notification_handler)

# Manter conectado
import time
while True:
    time.sleep(1)
```

### Liquid Wallet Kit (LWK)

**O que é**: Toolkit modular em Rust para Liquid

**Instalação**:

```toml
# Cargo.toml
[dependencies]
lwk = "0.4"
```

**Exemplo - Wallet básico**:

```rust
use lwk::{Wallet, Network, Mnemonic};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Criar wallet
    let mnemonic = Mnemonic::generate()?;
    let wallet = Wallet::new(Network::Liquid, &mnemonic)?;

    // Gerar endereço
    let address = wallet.get_address(0)?;
    println!("Endereço: {}", address);

    // Conectar a servidor Electrum
    let electrum_url = "blockstream.info:995";
    wallet.connect(electrum_url).await?;

    // Sincronizar
    wallet.sync().await?;

    // Obter saldo
    let balance = wallet.balance()?;
    println!("LBTC: {} sats", balance.lbtc);

    // Criar transação
    let recipient = "VJL...";
    let amount = 100_000; // 0.001 LBTC

    let tx = wallet.build_transaction()
        .add_recipient(recipient, amount)?
        .finish()?;

    // Assinar
    let signed = wallet.sign(&tx)?;

    // Broadcast
    let txid = wallet.broadcast(&signed).await?;
    println!("TX enviada: {}", txid);

    Ok(())
}
```

### Blockstream AMP API

**O que é**: API REST para emissão e gerenciamento de assets

**Autenticação**:

```bash
# Registrar em https://amp.blockstream.com/
# Obter API key
```

**Exemplo - Emitir asset via API**:

```python
import requests

AMP_API = "https://amp.blockstream.com/api/v1"
API_KEY = "your_api_key_here"

headers = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json"
}

# Criar asset
asset_data = {
    "name": "My Token",
    "ticker": "MTK",
    "precision": 8,
    "amount": 1000000,
    "domain": "mytoken.com"
}

response = requests.post(
    f"{AMP_API}/assets",
    headers=headers,
    json=asset_data
)

asset = response.json()
print(f"Asset ID: {asset['asset_id']}")
print(f"Reissuance token: {asset['token_id']}")

# Consultar asset
asset_id = "abc123..."
response = requests.get(
    f"{AMP_API}/assets/{asset_id}",
    headers=headers
)

print(response.json())

# Listar todos assets da conta
response = requests.get(
    f"{AMP_API}/assets",
    headers=headers
)

for asset in response.json()['assets']:
    print(f"{asset['ticker']}: {asset['asset_id']}")
```

### Libwally - Primitivas Criptográficas

**O que é**: Biblioteca de baixo nível para operações criptográficas

```python
import wallycore as wally

# Inicializar
wally.init(0)
wally.secp_randomize(os.urandom(32))

# Gerar mnemonic
mnemonic = wally.bip39_mnemonic_from_bytes(os.urandom(16))
print(f"Mnemonic: {mnemonic}")

# Derivar seed
seed = wally.bip39_mnemonic_to_seed(mnemonic, "")

# Criar chaves
master_key = wally.bip32_key_from_seed(
    seed,
    wally.BIP32_VER_MAIN_PRIVATE,
    0
)

# Derivar chave filho (m/84'/1'/0'/0/0)
path = [
    wally.bip32_initial_hardened_child + 84,
    wally.bip32_initial_hardened_child + 1,
    wally.bip32_initial_hardened_child + 0,
    0,
    0
]

child_key = master_key
for index in path:
    child_key = wally.bip32_key_from_parent(
        child_key,
        index,
        wally.BIP32_FLAG_KEY_PRIVATE
    )

# Obter endereço
pubkey = wally.bip32_key_get_pub_key(child_key)
address = wally.elements_pegout_script_from_bytes(
    pubkey,
    wally.WALLY_SCRIPT_SHA256,
    wally.ELEMENTS_PEGOUT_VERSION_1
)

print(f"Endereço: {address}")
```

---

## D.9 Simplicity - Smart Contracts no Liquid

### Introdução ao Simplicity

**O que é**:
- Linguagem de programação de baixo nível para smart contracts
- Projetada para **verificação formal**
- Alternativa ao Bitcoin Script
- Live em Liquid mainnet desde julho 2024

**Diferenças vs Solidity**:

| Aspecto | Solidity (Ethereum) | Simplicity (Liquid) |
|---------|---------------------|---------------------|
| **Paradigma** | Imperativo, OOP | Funcional, combinators |
| **Estado** | Global (account-based) | UTXO (stateless) |
| **Reentrância** | 🔴 Possível | 🟢 Impossível (UTXO) |
| **Verificação** | 🟡 Manual | 🟢 Formal (built-in) |
| **Expressividade** | 🟢 Alta | 🟡 Média |
| **Segurança** | 🟡 Depende do dev | 🟢 Por design |

### SimplicityHL (High-Level Language)

**Sintaxe similar a Rust**:

```rust
// ========================================
// EXEMPLO: Multisig 2-of-3
// ========================================

fn main() {
    // Pubkeys dos signatários
    let pk1: Pubkey = jet::pk_from_bytes([0x02, 0xab, ...]);
    let pk2: Pubkey = jet::pk_from_bytes([0x03, 0xcd, ...]);
    let pk3: Pubkey = jet::pk_from_bytes([0x02, 0xef, ...]);

    // Obter assinaturas da transação
    let sig1: Signature = jet::sig_from_stack();
    let sig2: Signature = jet::sig_from_stack();

    // Verificar assinaturas
    let valid1 = jet::bip340_verify(pk1, sig1);
    let valid2 = jet::bip340_verify(pk2, sig2);
    let valid3 = jet::bip340_verify(pk3, sig2);

    // Pelo menos 2 de 3 devem ser válidas
    let count = (valid1 as u8) + (valid2 as u8) + (valid3 as u8);
    assert!(count >= 2);
}
```

**Vault com timelock**:

```rust
// Vault que permite:
// - Owner gastar imediatamente
// - Recovery key após 1000 blocos

fn main() {
    let owner_pk: Pubkey = jet::pk_from_bytes([...]);
    let recovery_pk: Pubkey = jet::pk_from_bytes([...]);

    let sig: Signature = jet::sig_from_stack();

    // Path 1: Owner assina
    let owner_valid = jet::bip340_verify(owner_pk, sig);

    // Path 2: Recovery key + timelock
    let current_height = jet::current_block_height();
    let lock_height: u32 = 1000;
    let timelock_expired = current_height >= lock_height;
    let recovery_valid = jet::bip340_verify(recovery_pk, sig);

    // Aceita se owner OU (recovery E timelock expirado)
    assert!(owner_valid || (recovery_valid && timelock_expired));
}
```

**Covenant (restrição de gasto)**:

```rust
// Permite gastar apenas para endereços whitelist

fn main() {
    // Whitelist de destinatários permitidos
    let allowed1: [u8; 32] = [...];
    let allowed2: [u8; 32] = [...];

    // Obter output da transação
    let output_script = jet::output_script_hash();

    // Verificar se é um dos permitidos
    let is_allowed =
        jet::eq_256(output_script, allowed1) ||
        jet::eq_256(output_script, allowed2);

    assert!(is_allowed);
}
```

### Jets - Operações Built-in

**Categoria: Verificação de assinatura**:

```rust
// BIP-340 (Schnorr)
jet::bip340_verify(pubkey: Pubkey, sig: Signature) -> bool

// ECDSA
jet::ecdsa_verify(pubkey: Pubkey, sig: Signature) -> bool

// Recuperar pubkey de assinatura
jet::pubkey_recover(sig: Signature, msg_hash: u256) -> Pubkey
```

**Categoria: Hash**:

```rust
jet::sha256(data: [u8]) -> u256
jet::sha256_block(block: [u8; 64]) -> u256

jet::hash160(data: [u8]) -> u160  // SHA256 + RIPEMD160
```

**Categoria: Aritmética**:

```rust
jet::add_32(a: u32, b: u32) -> u32
jet::subtract_64(a: u64, b: u64) -> u64
jet::multiply_32(a: u32, b: u32) -> u64
jet::divide_32(a: u32, b: u32) -> (u32, u32)  // (quotient, remainder)

jet::max_32(a: u32, b: u32) -> u32
jet::min_64(a: u64, b: u64) -> u64
```

**Categoria: Comparação**:

```rust
jet::eq_32(a: u32, b: u32) -> bool
jet::lt_64(a: u64, b: u64) -> bool  // Less than
jet::le_32(a: u32, b: u32) -> bool  // Less or equal
```

**Categoria: Blockchain**:

```rust
jet::current_block_height() -> u32
jet::current_block_time() -> u32

jet::output_value(index: u32) -> u64
jet::output_script_hash(index: u32) -> u256
jet::output_asset(index: u32) -> u256  // Liquid-specific!

jet::tx_lock_time() -> u32
```

### Compilação e Deploy

**Setup**:

```bash
# Instalar compilador Simplicity
git clone https://github.com/BlockstreamResearch/simfony
cd simfony
cargo install --path .

# Verificar instalação
simfony --version
```

**Compilar contrato**:

```bash
# vault.simfony
simfony compile vault.simfony -o vault.simp

# Output: bytecode Simplicity
# Pode ser usado em transação Liquid
```

**Criar transação com Simplicity**:

```javascript
// Via elementos-cli (conceitual)
const simplicityBytecode = fs.readFileSync('vault.simp', 'hex');

const tx = await elements.createrawtransaction(
  [{ txid: utxo.txid, vout: utxo.vout }],
  { [recipientAddress]: amount }
);

// Adicionar Simplicity witness
const txWithWitness = await elements.setwitness(
  tx,
  0,  // input index
  simplicityBytecode
);

// Assinar e enviar
const signed = await elements.signrawtransactionwithwallet(txWithWitness);
const txid = await elements.sendrawtransaction(signed.hex);
```

### Verificação Formal

**Por que importante**:

```rust
// Solidity - Difícil de provar formalmente
contract Vault {
    mapping(address => uint256) balances;

    // Bug: reentrância possível
    function withdraw() external {
        uint256 amount = balances[msg.sender];
        (bool success, ) = msg.sender.call{value: amount}("");
        balances[msg.sender] = 0;  // ❌ Tarde demais
    }
}

// Simplicity - Reentrância impossível (UTXO)
fn withdraw() {
    // UTXO é consumido completamente
    // Novo output criado
    // Impossível chamar "novamente" o mesmo UTXO
}
```

**Ferramentas de verificação**:

```bash
# Verificar propriedades do contrato
simfony verify vault.simfony \
  --property "output_value >= input_value" \
  --property "timelock_respected"

# Output:
# ✅ All properties verified
# ⏱️ Verification time: 1.23s
```

---

## D.10 Desenvolvendo Wallets para Liquid

### Arquitetura de Wallet Liquid

**Componentes**:

```
┌─────────────────────────────────────────────┐
│              Liquid Wallet                  │
├─────────────────────────────────────────────┤
│                                             │
│  ┌────────────────────────────────────┐    │
│  │         UI Layer (React)           │    │
│  └────────────┬───────────────────────┘    │
│               ↓                             │
│  ┌────────────────────────────────────┐    │
│  │    Business Logic (TypeScript)     │    │
│  │  • Key management                  │    │
│  │  • TX building                     │    │
│  │  • Blinding                        │    │
│  └────────────┬───────────────────────┘    │
│               ↓                             │
│  ┌────────────────────────────────────┐    │
│  │       SDK Layer (GDK/LWK)          │    │
│  └────────────┬───────────────────────┘    │
│               ↓                             │
│  ┌────────────────────────────────────┐    │
│  │    Liquid Network (Electrum)       │    │
│  └────────────────────────────────────┘    │
│                                             │
└─────────────────────────────────────────────┘
```

### Wallet Básico - Código Completo

**1. Setup e derivação de chaves**:

```typescript
// wallet.ts
import { Mnemonic, BIP32, Network } from 'liquidjs-lib';
import * as bip39 from 'bip39';

export class LiquidWallet {
  private mnemonic: string;
  private seed: Buffer;
  private network: Network;

  constructor(network: 'liquid' | 'testnet' = 'liquid') {
    this.network = network === 'liquid'
      ? Network.LIQUID
      : Network.TESTNET;
  }

  // Criar novo wallet
  async create(): Promise<string> {
    // Gerar mnemonic BIP39
    this.mnemonic = bip39.generateMnemonic(256); // 24 palavras
    this.seed = await bip39.mnemonicToSeed(this.mnemonic);

    return this.mnemonic;
  }

  // Restaurar de mnemonic
  async restore(mnemonic: string): Promise<void> {
    if (!bip39.validateMnemonic(mnemonic)) {
      throw new Error('Invalid mnemonic');
    }

    this.mnemonic = mnemonic;
    this.seed = await bip39.mnemonicToSeed(mnemonic);
  }

  // Derivar chave (BIP84 - Native SegWit)
  deriveKey(account: number, change: number, index: number) {
    const root = BIP32.fromSeed(this.seed, this.network);

    // m/84'/1'/account'/change/index
    // 1 = Liquid coin type
    const path = `m/84'/1'/${account}'/${change}/${index}`;

    const child = root.derivePath(path);

    return {
      privateKey: child.privateKey,
      publicKey: child.publicKey,
      path: path
    };
  }

  // Gerar endereço confidential
  getAddress(account: number = 0, index: number = 0) {
    const key = this.deriveKey(account, 0, index);

    // Gerar blinding key
    const blindingKey = this.deriveKey(account, 1, index);

    // Criar confidential address
    const payment = payments.p2wpkh({
      pubkey: key.publicKey,
      network: this.network,
      blindkey: blindingKey.publicKey
    });

    return {
      address: payment.confidentialAddress,
      blindingKey: blindingKey.privateKey,
      publicKey: key.publicKey,
      path: key.path
    };
  }
}
```

**2. Gerenciamento de UTXOs**:

```typescript
// utxo-manager.ts
import { Electrum } from 'electrum-client';

export class UTXOManager {
  private electrum: Electrum;

  constructor(electrumUrl: string) {
    this.electrum = new Electrum(electrumUrl);
  }

  async connect() {
    await this.electrum.connect();
  }

  // Obter UTXOs de um endereço
  async getUTXOs(address: string): Promise<UTXO[]> {
    const scriptHash = this.addressToScriptHash(address);
    const utxos = await this.electrum.blockchain_scripthash_listunspent(
      scriptHash
    );

    return utxos.map(utxo => ({
      txid: utxo.tx_hash,
      vout: utxo.tx_pos,
      value: utxo.value,
      height: utxo.height,
      asset: utxo.asset,
      assetBlinder: utxo.assetblinder,
      valueBlinder: utxo.valueblinder
    }));
  }

  // Calcular saldo
  async getBalance(addresses: string[]): Promise<Balance> {
    const balance: Balance = {
      lbtc: 0,
      assets: {}
    };

    for (const address of addresses) {
      const utxos = await this.getUTXOs(address);

      for (const utxo of utxos) {
        if (utxo.asset === LBTC_ASSET_ID) {
          balance.lbtc += utxo.value;
        } else {
          balance.assets[utxo.asset] =
            (balance.assets[utxo.asset] || 0) + utxo.value;
        }
      }
    }

    return balance;
  }

  // Helpers
  private addressToScriptHash(address: string): string {
    const script = address.toOutputScript();
    const hash = crypto.sha256(script);
    return hash.reverse().toString('hex');
  }
}
```

**3. Construção de transações**:

```typescript
// tx-builder.ts
import { Transaction, Psbt } from 'liquidjs-lib';

export class TransactionBuilder {
  private wallet: LiquidWallet;
  private utxoManager: UTXOManager;

  async buildTransaction(params: TxParams): Promise<Psbt> {
    const { to, amount, asset, feeRate } = params;

    // 1. Selecionar UTXOs
    const utxos = await this.selectUTXOs(amount, asset);

    // 2. Criar PSBT
    const psbt = new Psbt({ network: this.wallet.network });

    // 3. Adicionar inputs
    for (const utxo of utxos) {
      psbt.addInput({
        hash: utxo.txid,
        index: utxo.vout,
        witnessUtxo: {
          script: utxo.scriptPubKey,
          value: utxo.value,
          asset: utxo.asset,
          nonce: utxo.nonce
        }
      });
    }

    // 4. Adicionar outputs

    // Output principal (confidential)
    const recipientBlindingPubkey = this.extractBlindingKey(to);

    psbt.addOutput({
      script: address.toOutputScript(to),
      value: amount,
      asset: asset,
      nonce: Buffer.concat([
        Buffer.from([0x01]), // Prefix
        recipientBlindingPubkey
      ])
    });

    // Change output (troco)
    const totalInput = utxos.reduce((sum, u) => sum + u.value, 0);
    const change = totalInput - amount - fee;

    if (change > 0) {
      const changeAddr = this.wallet.getAddress(0, this.changeIndex++);

      psbt.addOutput({
        script: address.toOutputScript(changeAddr.address),
        value: change,
        asset: asset,
        nonce: Buffer.concat([
          Buffer.from([0x01]),
          changeAddr.blindingKey
        ])
      });
    }

    // Fee output (explícito no Liquid)
    psbt.addOutput({
      script: Buffer.alloc(0),  // Empty script
      value: fee,
      asset: asset,
      nonce: Buffer.from([0x00])  // Unblinded
    });

    return psbt;
  }

  // Blind transaction (ocultar valores e assets)
  async blindTransaction(psbt: Psbt): Promise<Psbt> {
    // Usar GDK ou Libwally para blinding
    const blinded = await gdk.blindTransaction(psbt);
    return blinded;
  }

  // Assinar transação
  async signTransaction(psbt: Psbt): Promise<Psbt> {
    for (let i = 0; i < psbt.data.inputs.length; i++) {
      const input = psbt.data.inputs[i];
      const keyPair = this.wallet.getKeyPair(input.path);

      psbt.signInput(i, keyPair);
    }

    psbt.finalizeAllInputs();
    return psbt;
  }

  // Broadcast
  async broadcastTransaction(psbt: Psbt): Promise<string> {
    const tx = psbt.extractTransaction();
    const txHex = tx.toHex();

    const txid = await this.utxoManager.broadcast(txHex);
    return txid;
  }
}
```

**4. Interface React**:

```tsx
// App.tsx
import React, { useState, useEffect } from 'react';
import { LiquidWallet } from './wallet';

export function App() {
  const [wallet, setWallet] = useState<LiquidWallet | null>(null);
  const [balance, setBalance] = useState(0);
  const [address, setAddress] = useState('');

  // Criar wallet
  const createWallet = async () => {
    const w = new LiquidWallet('liquid');
    const mnemonic = await w.create();

    // IMPORTANTE: Usuário deve guardar mnemonic
    alert(`Guarde suas palavras:\n\n${mnemonic}`);

    setWallet(w);
    const addr = w.getAddress(0, 0);
    setAddress(addr.address);
  };

  // Atualizar saldo
  useEffect(() => {
    if (!wallet) return;

    const updateBalance = async () => {
      const utxoManager = new UTXOManager('blockstream.info:995');
      await utxoManager.connect();

      const addresses = [address];
      const bal = await utxoManager.getBalance(addresses);

      setBalance(bal.lbtc);
    };

    updateBalance();
    const interval = setInterval(updateBalance, 30000); // 30s

    return () => clearInterval(interval);
  }, [wallet, address]);

  // Enviar transação
  const sendTransaction = async (to: string, amount: number) => {
    if (!wallet) return;

    const builder = new TransactionBuilder(wallet);

    try {
      // Construir
      let psbt = await builder.buildTransaction({
        to,
        amount,
        asset: LBTC_ASSET_ID,
        feeRate: 0.1
      });

      // Blind
      psbt = await builder.blindTransaction(psbt);

      // Assinar
      psbt = await builder.signTransaction(psbt);

      // Enviar
      const txid = await builder.broadcastTransaction(psbt);

      alert(`Transação enviada: ${txid}`);
    } catch (error) {
      alert(`Erro: ${error.message}`);
    }
  };

  return (
    <div>
      <h1>Liquid Wallet</h1>

      {!wallet ? (
        <button onClick={createWallet}>Criar Wallet</button>
      ) : (
        <div>
          <p>Endereço: {address}</p>
          <p>Saldo: {balance / 100000000} LBTC</p>

          <SendForm onSend={sendTransaction} />
        </div>
      )}
    </div>
  );
}
```

---

## D.11 Emissão de Assets - Do Código ao Registry

### Processo Completo de Emissão

**1. Emitir via elements-cli**:

```bash
#!/bin/bash

# ========================================
# ASSET ISSUANCE - PRODUCTION READY
# ========================================

# Parâmetros
ASSET_NAME="My Stablecoin"
TICKER="MSTB"
INITIAL_SUPPLY=10000000  # 100M tokens (8 decimals)
PRECISION=8
DOMAIN="mystablecoin.com"

# Passo 1: Emitir asset
echo "Emitindo $ASSET_NAME..."

ISSUANCE=$(elements-cli issueasset $INITIAL_SUPPLY 1 false)

ASSET_ID=$(echo $ISSUANCE | jq -r '.asset')
TOKEN_ID=$(echo $ISSUANCE | jq -r '.token')
TX_ID=$(echo $ISSUANCE | jq -r '.txid')

echo "✅ Asset emitido!"
echo "Asset ID: $ASSET_ID"
echo "Token ID: $TOKEN_ID"
echo "TX: $TX_ID"

# Passo 2: Criar contract hash (PDF de termos legais)
CONTRACT_FILE="terms_and_conditions.pdf"
CONTRACT_HASH=$(sha256sum $CONTRACT_FILE | awk '{print $1}')

echo "Contract hash: $CONTRACT_HASH"

# Passo 3: Preparar registry metadata
cat > asset_metadata.json <<EOF
{
  "asset_id": "$ASSET_ID",
  "contract": {
    "entity": {
      "domain": "$DOMAIN"
    },
    "issuer_pubkey": "$(elements-cli getaddressinfo $(elements-cli getnewaddress) | jq -r '.pubkey')",
    "name": "$ASSET_NAME",
    "precision": $PRECISION,
    "ticker": "$TICKER",
    "version": 0
  },
  "contract_hash": "$CONTRACT_HASH"
}
EOF

echo "✅ Metadata preparado: asset_metadata.json"
```

**2. Proof de domínio**:

```bash
# Gerar proof
cat > liquid-asset-proof-${ASSET_ID} <<EOF
Authorize linking the domain name $DOMAIN to the Liquid asset $ASSET_ID
EOF

# Assinar com chave do issuer
SIGNATURE=$(elements-cli signmessage \
  $(elements-cli getnewaddress) \
  "$(cat liquid-asset-proof-${ASSET_ID})")

cat > .well-known/liquid-asset-proof-${ASSET_ID} <<EOF
{
  "asset_id": "$ASSET_ID",
  "contract": $(cat asset_metadata.json | jq '.contract'),
  "proof": "$SIGNATURE"
}
EOF

echo "✅ Hospedar em: https://$DOMAIN/.well-known/liquid-asset-proof-$ASSET_ID"
```

**3. Submeter ao Registry**:

```javascript
// submit-to-registry.js
const axios = require('axios');
const fs = require('fs');

const REGISTRY_API = 'https://assets.blockstream.info/';

async function submitAsset() {
  const metadata = JSON.parse(fs.readFileSync('asset_metadata.json'));

  // Submeter
  const response = await axios.post(
    `${REGISTRY_API}api/asset`,
    metadata,
    {
      headers: { 'Content-Type': 'application/json' }
    }
  );

  console.log('✅ Asset registrado no registry!');
  console.log('URL:', `${REGISTRY_API}asset/${metadata.asset_id}`);

  return response.data;
}

submitAsset().catch(console.error);
```

**4. Verificar no explorer**:

```bash
# Blockstream Liquid Explorer
open "https://blockstream.info/liquid/asset/$ASSET_ID"

# Ou via API
curl "https://blockstream.info/liquid/api/asset/$ASSET_ID"

# Output:
# {
#   "asset_id": "abc123...",
#   "name": "My Stablecoin",
#   "ticker": "MSTB",
#   "precision": 8,
#   "entity": {
#     "domain": "mystablecoin.com"
#   },
#   "chain_stats": {
#     "issued_amount": 10000000000000000,
#     "burned_amount": 0,
#     "tx_count": 1
#   }
# }
```

### Re-emissão (Minting Adicional)

**Usando reissuance token**:

```bash
# Verificar se tem reissuance token
elements-cli listissuances

# Output mostra token_id

# Re-emitir mais tokens
ADDITIONAL_AMOUNT=5000000  # 50M mais

elements-cli reissueasset \
  "$ASSET_ID" \
  $ADDITIONAL_AMOUNT

# Agora supply total = 150M
```

### Queima (Burn)

**Enviar para endereço OP_RETURN**:

```bash
# Criar TX de burn
elements-cli sendtoaddress \
  "" \
  100000 \
  "" \
  "" \
  false \
  false \
  1 \
  "UNSET" \
  false \
  "$ASSET_ID" \
  true  # ← burn flag

# Tokens são destruídos permanentemente
```

---

## D.12 Comparativo - Liquid vs Lightning vs Rootstock

### Tabela Comparativa Detalhada

| Aspecto | **Liquid Network** | **Lightning Network** | **Rootstock (RSK)** |
|---------|-------------------|----------------------|---------------------|
| **Tipo** | Sidechain federada | Payment channels (L2) | Sidechain merge-mined |
| **Lançamento** | 2018 | 2018 | 2018 |
| **TVL (2024)** | ~$500M | ~$200M (5k BTC) | ~$150M |
| **Confirmação** | ~2 min (final) | Instantâneo | ~30 seg |
| **TPS** | ~100-200 | Milhares (off-chain) | ~10-20 |
| **Consenso** | Strong Federation (15 functionaries) | Nenhum (P2P channels) | Merged mining (PoW) |
| **Custódia BTC** | Multisig 11-of-15 | Usuário (Lightning wallet) | Federated peg (Powpeg) |
| **Privacidade** | 🟢 Confidential Transactions | 🟢 Onion routing | 🔴 Pública (como Bitcoin) |
| **Smart Contracts** | 🟢 Simplicity | 🔴 Não | 🟢 EVM (Solidity) |
| **Assets nativos** | 🟢 Issued Assets | 🔴 Não (apenas BTC) | 🟡 Via tokens ERC-20 |
| **Custos** | 🟢 Baixo e fixo | 🟢 Mínimo | 🟡 Médio (gas) |
| **Descentralização** | 🟡 15 entidades | 🟢 Qualquer um | 🟡 Mineradores Bitcoin |
| **Uso principal** | Trading, stablecoins, securities | Pagamentos rápidos | DeFi, dApps |
| **Complexidade** | 🟡 Média | 🔴 Alta (canais, liquidez) | 🟢 Baixa (igual Ethereum) |

### Quando Usar Cada Um

**Liquid Network** ✅:
- Emitir stablecoins ou securities
- Trading entre exchanges (privado e rápido)
- Transações confidenciais de alto valor
- Smart contracts com verificação formal
- Atomic swaps entre assets

**Lightning Network** ✅:
- Micropagamentos frequentes
- Streaming de pagamentos
- Aplicações que precisam de finality instantânea
- Custos absolutamente mínimos
- Máxima descentralização

**Rootstock** ✅:
- Portar dApps de Ethereum para Bitcoin
- DeFi com compatibilidade EVM
- Usar Solidity e ferramentas Ethereum
- Precisar de ecosystem maduro (oracles, DEXs, etc.)

### Comparação de Código

**Transferir valor**:

```javascript
// LIQUID
elements.sendtoaddress(
  "VJL...",
  0.1,     // 0.1 LBTC
  "",
  "",
  false,
  false,
  1,
  "UNSET",
  false,
  LBTC_ASSET_ID
);
// Confirmação: ~2 min
// Privacidade: ✅ Confidential
// Custo: ~0.00001 LBTC

// LIGHTNING
lnd.sendPayment({
  payment_request: "lnbc...",  // Invoice
  amt: 10000  // satoshis
});
// Confirmação: instantâneo
// Privacidade: ✅ Roteado
// Custo: ~0.000001 BTC (routing fees)

// ROOTSTOCK
web3.eth.sendTransaction({
  from: "0xabc...",
  to: "0xdef...",
  value: web3.utils.toWei("0.1", "ether"),
  gas: 21000
});
// Confirmação: ~30 seg
// Privacidade: ❌ Pública
// Custo: 21000 * gasPrice (em RBTC)
```

**Smart contract**:

```solidity
// LIQUID (Simplicity)
fn vault() {
    let owner_pk = jet::pk_from_bytes([...]);
    let sig = jet::sig_from_stack();
    assert!(jet::bip340_verify(owner_pk, sig));
}
// Verificação formal: ✅
// Reentrância: ❌ Impossível (UTXO)

// LIGHTNING
// ❌ Não suporta smart contracts arbitrários
// Apenas HTLCs (Hash Time-Locked Contracts)

// ROOTSTOCK (Solidity)
contract Vault {
    address owner;

    function withdraw() external {
        require(msg.sender == owner);
        payable(owner).transfer(address(this).balance);
    }
}
// Compatibilidade: ✅ 100% EVM
// Reentrância: ⚠️ Depende do dev
```

---

## D.13 Casos de Uso e Ecossistema

### Trading e Exchanges

**Problema resolvido**:
- Transfers entre exchanges são lentos (Bitcoin) e públicos
- Custos altos em períodos de congestão

**Solução Liquid**:

```
Exchange A  ─────────────►  Exchange B
   (Bitfinex)    2 min      (BTSE)
                Confidential
                Baixo custo
```

**Exchanges no Liquid** (alguns):
- Bitfinex
- BTSE
- Sideswap (DEX nativa Liquid)
- Hodl Hodl (P2P)
- Bisq (suporte Liquid)

**Exemplo - Arbitragem rápida**:

```javascript
// Arbitrage bot
class LiquidArbitrage {
  async findOpportunity() {
    // Preço LBTC/USDT em diferentes exchanges
    const bitfinexPrice = await this.getBitfinexPrice();
    const btsePrice = await this.getBTSEPrice();

    if (bitfinexPrice < btsePrice * 0.99) {
      // Oportunidade: comprar na Bitfinex, vender na BTSE

      // 1. Comprar LBTC na Bitfinex
      await this.buyLBTC('bitfinex', 1.0);

      // 2. Transferir para BTSE (2 minutos, confidencial)
      const txid = await this.withdrawLBTC(
        'bitfinex',
        BTSE_LIQUID_ADDRESS,
        1.0
      );

      // 3. Aguardar confirmação
      await this.waitConfirmation(txid, 2);

      // 4. Vender na BTSE
      await this.sellLBTC('btse', 1.0);

      // Lucro: (btsePrice - bitfinexPrice) * 1.0 - fees
    }
  }
}
```

### Stablecoins Institucionais

**Tether (USDT)**:

```javascript
// Liquidez entre chains via Liquid
class CrossChainUSDT {
  async ethereumToLiquid(amount: number) {
    // 1. Usuário tem USDT na Ethereum
    // 2. Deposita em exchange que suporta ambas

    await bitfinex.deposit('USDT', 'ethereum', amount);

    // 3. Internamente: exchange queima ERC-20 e emite Liquid

    // 4. Saca para endereço Liquid (confidential)
    const liquidAddr = "VJL...";
    await bitfinex.withdraw('USDT', 'liquid', liquidAddr, amount);

    // ✅ USDT agora no Liquid
    // - Privado (valores ocultos)
    // - Rápido (2 min)
    // - Barato
  }
}
```

**Volume USDT Liquid** (2024):
- ~$100M em circulação
- Usado por: traders, OTC desks, institucionais

### Securities e Tokenização

**Blockstream Mining Notes (BMN)**:
- Security tokens representando hashrate de mineração
- Emitidos no Liquid
- Compliantes com regulação

**Vantagens**:

```typescript
// Security token no Liquid
interface SecurityToken {
  asset_id: string;

  // Metadata compliance
  metadata: {
    name: "Blockstream Mining Note 2024";
    type: "security";
    jurisdiction: "CA";  // Canadá
    regulatory_approval: "OSC-2024-123";
  };

  // Transferências restritas
  transfer_restrictions: {
    kyc_required: true;
    accredited_investors_only: true;
    lock_period: 365 * 24 * 60 * 60;  // 1 ano
  };

  // Dividendos automáticos
  dividends: {
    asset: LBTC_ASSET_ID;
    frequency: "monthly";
    distribution_contract: "simplicity_hash_here";
  };
}
```

**Implementação - Distribuição de dividendos**:

```rust
// Simplicity - Contrato de dividendos
fn distribute_dividends() {
    // Lista de holders (off-chain, provido como witness)
    let holders: Vec<(Pubkey, u64)> = jet::witness_holders();

    // Total de dividendos a distribuir
    let total_dividends: u64 = 1000000;  // 0.01 LBTC

    // Total supply do security token
    let total_supply: u64 = 10000000;

    // Para cada holder
    for (holder_pk, balance) in holders {
        // Calcular dividendo proporcional
        let dividend = (balance * total_dividends) / total_supply;

        // Criar output
        let output = create_output(holder_pk, dividend, LBTC_ASSET_ID);

        // Adicionar à transação
        add_output(output);
    }

    assert!(verify_total_outputs() == total_dividends);
}
```

### DeFi no Liquid

**SideSwap** - DEX nativa:

```typescript
// Atomic swap via SideSwap
class SideSwapDEX {
  async swapUSDTforLBTC(amountUSDT: number) {
    // 1. Criar proposta de swap
    const proposal = {
      send_asset: USDT_ASSET_ID,
      send_amount: amountUSDT * 1e8,
      recv_asset: LBTC_ASSET_ID,
      recv_amount: 0,  // Será preenchido pelo market maker
      price: null      // Aceita melhor preço
    };

    // 2. Market maker responde
    const quote = await sideswap.getQuote(proposal);

    // Output:
    // {
    //   recv_amount: 0.003 LBTC,
    //   price: 0.003 / (amountUSDT / 100000),
    //   expires_at: timestamp + 30
    // }

    // 3. Aceitar e executar (atomic)
    const swap = await sideswap.executeSwap(quote);

    // ✅ USDT → LBTC instantâneo
    // - Sem custódia (atomic)
    // - Confidential

    return swap.txid;
  }
}
```

**Hodl Hodl** - P2P Lending:

```javascript
// Emprestar USDT, usar LBTC como colateral
class LiquidLending {
  async createLoanOffer() {
    const offer = {
      lend: {
        asset: USDT_ASSET_ID,
        amount: 10000  // $10k USDT
      },
      collateral: {
        asset: LBTC_ASSET_ID,
        amount: 0.5,  // 0.5 LBTC (~150% colateral)
        ltv: 0.66     // Loan-to-Value 66%
      },
      terms: {
        duration: 30 * 24 * 60 * 60,  // 30 dias
        interest_rate: 0.05,           // 5% APR
        liquidation_ltv: 0.80          // Liquidação se LTV > 80%
      }
    };

    // Criar multisig 2-of-2 (lender + borrower)
    const multisig = await this.createMultisig(
      offer.lend.amount,
      offer.collateral.amount
    );

    return { offer, multisig };
  }

  // Smart contract Simplicity para liquidação
  async liquidationContract() {
    // Se preço LBTC cair muito, permite liquidação
    // Implementado em Simplicity ou via oracle + multisig
  }
}
```

---

## D.14 Segurança e Trade-offs

### Modelo de Confiança

**Liquid não é trustless** como Bitcoin:

```
Bitcoin:
  ✅ Trustless: Qualquer um pode minerar
  ✅ Descentralizado: Milhares de nodes
  ✅ Censorship-resistant: Difícil censurar TX

Liquid:
  ⚠️ Trust-minimized: Confia em 11 de 15 functionaries
  ⚠️ Federado: 15 entidades controlam rede
  ⚠️ Pode censurar: Functionaries podem recusar TXs
```

**Mitigações**:

1. **Functionaries geograficamente distribuídos**:
   - América do Norte, Europa, Ásia
   - Diferentes jurisdições legais

2. **HSMs customizados**:
   - Chaves nunca expostas
   - Software verificável

3. **Código open-source**:
   - Elements é auditável
   - Functionary code é público

4. **Multisig 11-of-15**:
   - Precisa comprometer 11 entidades
   - Melhor que custódia única

### Riscos

**1. Centralização da Federação**:

```javascript
// Cenário: 11+ functionaries comprometidos
const risks = {
  doublespend: {
    description: "Assinar blocos conflitantes",
    mitigation: "HSMs impedem, mas teoricamente possível",
    probability: "Muito baixa"
  },

  censorship: {
    description: "Recusar incluir certas TXs",
    mitigation: "Reputação das entidades, code of conduct",
    probability: "Baixa mas possível"
  },

  peg_theft: {
    description: "Roubar BTC do multisig",
    mitigation: "HSMs, código open-source, auditorias",
    probability: "Muito baixa (nunca ocorreu)"
  }
};
```

**2. Bugs no Elements**:

```solidity
// Exemplo teórico
contract ElementsBug {
    // Se houver bug em Confidential Transactions
    // Poderia permitir inflation oculta

    function inflationBug() {
        // Criar commitment que não bate
        // Mas passa na verificação (bug)

        // ⚠️ Mitigação:
        // - Auditorias constantes
        // - Code review
        // - Testes extensivos
    }
}
```

**Histórico**: Liquid teve um fork acidental em 2023 (bug em validação), resolvido em horas.

**3. Confidential Transactions - Quebra criptográfica**:

```python
# Cenário: Discrete Log quebrado no futuro
class QuantumThreat:
    """
    Se computador quântico quebrar ECDLP:
    - Pedersen Commitments ficam expostos
    - Valores e assets revelados retroativamente
    """

    mitigation = {
        "Quantum-resistant cryptography": "Pesquisa ativa",
        "Upgrade path": "Hard fork se necessário",
        "Timeframe": "Provavelmente décadas"
    }
```

### Melhores Práticas

**Para usuários**:

```javascript
// 1. Não deixar grandes quantias por muito tempo
const recommendation = {
  liquid_usage: "Trading, transferências rápidas",
  long_term_holding: "Bitcoin cold storage (trustless)",

  max_recommended: "Valor que você confiaria a um exchange"
};

// 2. Verificar functionaries periodicamente
async function auditFunctionaries() {
  const functionaries = await elements.getblockchaininfo();

  // Verificar se ainda são entidades confiáveis
  // Monitorar mudanças na federação
}

// 3. Entender unblinding
// Seus dados de TX podem ser revelados a:
// - Reguladores (se exchange for membro)
// - Você mesmo (backup de blinding keys)
// - Potencialmente outros (se compartilhar keys)
```

**Para desenvolvedores**:

```typescript
// 1. Sempre validar proofs
class SecurityChecks {
  async validateConfidentialTx(tx: Transaction) {
    // Verificar range proofs
    for (const output of tx.outputs) {
      const valid = await verifyRangeProof(
        output.commitment,
        output.rangeProof
      );

      if (!valid) {
        throw new Error("Invalid range proof");
      }
    }

    // Verificar que sum(inputs) = sum(outputs)
    const inputSum = sumCommitments(tx.inputs.map(i => i.commitment));
    const outputSum = sumCommitments(tx.outputs.map(o => o.commitment));

    if (!commitmentsEqual(inputSum, outputSum)) {
      throw new Error("Commitments don't balance");
    }
  }

  // 2. Backup de blinding keys
  async backupBlindingKeys(wallet: Wallet) {
    const keys = wallet.exportBlindingKeys();

    // Encriptar e guardar com segurança
    const encrypted = await encrypt(keys, userPassword);
    await secureStorage.save('blinding_keys_backup', encrypted);
  }

  // 3. Monitorar functionaries
  async monitorFederation() {
    const info = await elements.getblockchaininfo();

    // Alertar se mudanças
    if (info.functionaries !== EXPECTED_FUNCTIONARIES) {
      await alert('Federation composition changed!');
    }
  }
}
```

---

## 📖 Glossário Consolidado

**LBTC (Liquid Bitcoin)**
> Bitcoin "pegado" na Liquid Network através do mecanismo de peg bidirecional.
>
> **Paridade**: Sempre 1 LBTC = 1 BTC
>
> **Custódia**: Multisig 11-of-15 controlado pelos Watchmen

**Confidential Transactions**
> Tecnologia que oculta valores e tipos de asset em transações usando criptografia (Pedersen Commitments).
>
> **Trade-off**: Privacidade vs tamanho da TX (~20x maior)

**Strong Federations**
> Modelo de consenso onde grupo de entidades mutuamente desconfiadas (functionaries) operam a rede.
>
> **Analogia**: Como consórcio de empresas gerenciando blockchain juntas

**Functionaries**
> Entidades que operam a Liquid Network com dois papéis: Blocksigners (produzem blocos) e Watchmen (guardam BTC).
>
> **Quantidade**: 15 entidades
> **Threshold**: 11 de 15 para consenso

**HSM (Hardware Security Module)**
> Dispositivo de hardware que armazena chaves criptográficas e executa operações sem expor as chaves.
>
> **Analogia Web2**: Como AWS KMS mas em hardware dedicado

**Peg-In**
> Processo de mover BTC da mainnet Bitcoin para Liquid Network (vira LBTC).
>
> **Tempo**: ~17 horas (102 confirmações Bitcoin)

**Peg-Out**
> Processo de mover LBTC de volta para Bitcoin (vira BTC).
>
> **Tempo**: ~30-90 minutos
> **Limitação**: Apenas Federation Members podem iniciar

**Issued Assets**
> Tokens nativos criados no Liquid sem necessidade de smart contracts.
>
> **Vantagem**: Mesmo tratamento que LBTC (atomic swaps, multisig, etc.)

**Blinding**
> Processo criptográfico de ocultar valores e assets em transações Liquid.
>
> **Técnica**: Pedersen Commitments + Bulletproofs

**Blinding Key**
> Chave privada usada para "desblindear" (revelar) valores de transações confidenciais.
>
> **Uso**: Compliance, auditoria, backup

**Range Proof**
> Prova criptográfica de que um valor oculto está dentro de um intervalo (ex: 0 a 2^64) sem revelar o valor.
>
> **Tamanho**: ~700 bytes (com Bulletproofs)

**Elements**
> Plataforma blockchain open-source que é a base do Liquid Network.
>
> **Relação**: Como Bitcoin Core está para Bitcoin, Elements está para Liquid

**Simplicity**
> Linguagem de smart contracts de baixo nível para Liquid, projetada para verificação formal.
>
> **Status**: Live em mainnet desde julho 2024

**SimplicityHL / Simfony**
> Linguagem de alto nível (similar a Rust) que compila para Simplicity.
>
> **Objetivo**: Facilitar desenvolvimento de contratos verificáveis

**Asset Registry**
> Registro público de metadata de Issued Assets (nome, ticker, domínio emissor, etc.).
>
> **URL**: https://assets.blockstream.info/

**Atomic Swap**
> Troca entre dois assets (ex: LBTC ↔ USDT) que acontece completamente ou não acontece (atômica).
>
> **Vantagem no Liquid**: Nativo, sem necessidade de smart contract

**Unblinding**
> Processo de revelar valores e assets de uma transação confidencial usando blinding keys.
>
> **Caso de uso**: Auditoria, compliance, prova de pagamento

**GDK (Green Development Kit)**
> SDK multi-plataforma da Blockstream para criar wallets Liquid.
>
> **Linguagens**: Python, Java, Swift, Kotlin

**LWK (Liquid Wallet Kit)**
> Toolkit modular em Rust para construir aplicações Liquid.
>
> **Diferencial**: Mais modular e leve que GDK

**Blockstream AMP**
> API e plataforma para emissão e gerenciamento de assets no Liquid.
>
> **Uso**: Empresas que querem emitir securities, stablecoins, etc.

**Jets (Simplicity)**
> Operações built-in otimizadas na linguagem Simplicity.
>
> **Exemplos**: `jet::sha256`, `jet::bip340_verify`, `jet::add_32`

---

## 📚 Recursos e Documentação

### Documentação Oficial

**Essenciais** (SEMPRE atualizado):

1. **Liquid Network Docs** - https://docs.liquid.net/
   - Tutorial completo de desenvolvimento
   - API references
   - Building on Liquid guide

2. **Elements Project** - https://elementsproject.org/
   - Documentação técnica da plataforma Elements
   - Sidechain tutorial
   - Features em detalhes

3. **Blockstream Help Center** - https://help.blockstream.com/
   - Guias passo-a-passo
   - FAQs
   - Troubleshooting

### APIs e SDKs

4. **GDK Documentation** - https://github.com/Blockstream/gdk
   - Green Development Kit
   - Exemplos em Python, Java
   - Wallet development

5. **LWK Repository** - https://github.com/Blockstream/lwk
   - Liquid Wallet Kit (Rust)
   - Modular components
   - Modern architecture

6. **Blockstream AMP** - https://blockstream.com/amp/
   - Asset issuance platform
   - API documentation
   - Enterprise features

### Simplicity

7. **Simplicity GitHub** - https://github.com/BlockstreamResearch/simplicity
   - Spec completa da linguagem
   - Paper original
   - Implementação de referência

8. **Simfony Compiler** - https://github.com/BlockstreamResearch/simfony
   - High-level language
   - Examples
   - Web IDE

9. **Simplicity Whitepaper** - https://blockstream.com/simplicity.pdf
   - Russell O'Connor
   - Teoria e design

### Explorers e Ferramentas

10. **Blockstream Liquid Explorer** - https://blockstream.info/liquid/
    - Explorer oficial
    - Asset registry
    - API pública

11. **Mempool.space Liquid** - https://liquid.network/
    - Explorer alternativo
    - Estatísticas da rede
    - Fee estimator

12. **Nigiri** - https://github.com/vulpemventures/nigiri
    - Ambiente local de desenvolvimento
    - Docker-based
    - Elements + Bitcoin regtest

### Carteiras

13. **Blockstream Green** - https://blockstream.com/green/
    - Wallet oficial Blockstream
    - Suporte LBTC e Issued Assets
    - Mobile e Desktop

14. **Aqua Wallet** - https://aquawallet.io/
    - Foco em UX
    - Swap integrado (SideSwap)
    - Mobile-first

15. **SideSwap** - https://sideswap.io/
    - Wallet + DEX
    - Instant swaps
    - Market making

### Exchanges e Serviços

16. **Bitfinex** - https://www.bitfinex.com/
    - Exchange com suporte Liquid
    - USDT Liquid
    - Peg-in/peg-out

17. **BTSE** - https://www.btse.com/
    - Trading Liquid assets
    - OTC desk
    - API

18. **Hodl Hodl** - https://hodlhodl.com/
    - P2P lending
    - Non-custodial
    - LBTC e USDT Liquid

### Comunidade

**Discord**:
- Blockstream Community - https://discord.gg/blockstream

**Telegram**:
- Liquid Network - https://t.me/liquid_network

**Twitter/X**:
- @Liquid_BTC
- @Blockstream
- @SideSwap_io

**GitHub**:
- https://github.com/ElementsProject
- https://github.com/Blockstream

### Blogs e Updates

19. **Liquid Network Blog** - https://blog.liquid.net/
    - Announcements
    - Technical deep dives
    - Ecosystem updates

20. **Blockstream Blog** - https://blog.blockstream.com/
    - Company updates
    - Research posts
    - Simplicity updates

### Papers e Pesquisa

21. **Liquid Whitepaper** - https://blockstream.com/assets/downloads/pdf/liquid-whitepaper.pdf
    - Jonas Nick
    - Arquitetura completa
    - Strong Federations

22. **Confidential Transactions** - https://elementsproject.org/features/confidential-transactions
    - Gregory Maxwell
    - Matemática por trás
    - Implementação

23. **Strong Federations Paper** - https://blockstream.com/strong-federations.pdf
    - Modelo de consenso
    - Security analysis

### Cursos e Tutoriais

24. **Elements Code Tutorial** - https://elementsproject.org/elements-code-tutorial/
    - Tutorial interativo
    - Exemplos práticos
    - Do básico ao avançado

25. **Building on Liquid (Docs)** - https://docs.liquid.net/docs/building-on-liquid
    - Setup environment
    - First transaction
    - Asset issuance

### Ferramentas de Desenvolvimento

**CLI Tools**:
```bash
# Elements daemon e CLI
https://github.com/ElementsProject/elements

# Liquid testnet explorer
https://liquid.network/testnet/
```

**Libraries**:
```
# JavaScript
liquidjs-lib - https://github.com/vulpemventures/liquidjs-lib

# Python
python-elementstx - https://github.com/Simplexum/python-elementstx

# Rust
elements - https://github.com/ElementsProject/rust-elements
```

---

## 🎯 Próximos Passos

### Para Começar Hoje

**Nível Iniciante** (1-2 semanas):

1. **Setup ambiente**:
   ```bash
   # Instalar Elements testnet
   docker run -d blockstream/elementsd:latest -chain=liquidtestnet

   # Criar wallet
   elements-cli -chain=liquidtestnet createwallet "testwallet"

   # Obter endereço
   elements-cli -chain=liquidtestnet getnewaddress
   ```

2. **Obter fundos testnet**:
   - Usar faucet (buscar "liquid testnet faucet")
   - Explorar transações no explorer

3. **Primeira transação confidential**:
   - Enviar para outro endereço seu
   - Observar valores ocultos
   - Praticar unblinding

**Nível Intermediário** (2-4 semanas):

4. **Emitir seu primeiro asset**:
   ```bash
   # Emitir 1M tokens
   elements-cli issueasset 1000000 1 false

   # Registrar no registry (testnet)
   # Criar metadata JSON
   # Fazer proof de domínio
   ```

5. **Desenvolver wallet básico**:
   - Usar GDK ou LWK
   - Implementar send/receive
   - Gerenciar múltiplos assets

6. **Experimentar atomic swaps**:
   - Trocar LBTC por seu asset
   - Usar SideSwap API
   - Implementar swap próprio

**Nível Avançado** (1-2 meses):

7. **Smart contract Simplicity**:
   - Instalar Simfony
   - Escrever vault com timelock
   - Deploy em testnet

8. **Integração completa**:
   - Backend: API para aceitar pagamentos Liquid
   - Frontend: UI para wallet
   - Monitoring: Escutar eventos ZMQ

9. **Produção**:
   - Mainnet setup
   - Security audit
   - Lançar aplicação real

### Projetos Sugeridos

**DeFi**:
1. **Liquid Stablecoin**: Emitir stablecoin lastreada em BTC
2. **Lending Protocol**: P2P lending com Simplicity covenants
3. **DEX**: Atomic swap orderbook
4. **Yield Aggregator**: Otimizar retornos entre protocols

**Infraestrutura**:
5. **Payment Processor**: API para merchants aceitarem LBTC/USDT
6. **Bridge**: Facilitar peg-in/peg-out via interface web
7. **Explorer**: Block explorer customizado com analytics
8. **Wallet**: Wallet mobile com foco em UX

**Enterprise**:
9. **Security Token Platform**: Emissão de securities compliant
10. **Supply Chain**: Rastreamento de ativos com Issued Assets
11. **Settlement Layer**: Infraestrutura para exchanges
12. **Remittances**: Transferências internacionais privadas

### Roadmap de Aprendizado

```
Semana 1-2: Fundamentos
├─ Setup Elements
├─ Entender Confidential Transactions
├─ Primeira TX testnet
└─ Explorar Liquid explorer

Semana 3-4: Desenvolvimento Básico
├─ SDK (GDK ou LWK)
├─ Wallet simples
├─ Asset issuance
└─ Atomic swaps

Semana 5-8: Intermediário
├─ Simplicity basics
├─ Smart contracts simples
├─ Integração backend
└─ Testing extensivo

Semana 9-12: Avançado
├─ Aplicação full-stack
├─ Produção em mainnet
├─ Security hardening
└─ Lançamento público

Contínuo:
├─ Acompanhar updates Simplicity
├─ Participar comunidade
├─ Contribuir open-source
└─ Explorar novos use cases
```

### Manter-se Atualizado

**Monitorar**:
- [ ] Liquid Network blog (mensal)
- [ ] Blockstream announcements (semanal)
- [ ] GitHub releases (Elements, Simplicity)
- [ ] Discord/Telegram (diário para dúvidas)
- [ ] Twitter devs-chave (@Blockstream, etc.)

**Experimentar**:
- [ ] Testnet sempre que possível
- [ ] Contribuir para projetos open-source
- [ ] Participar de hackathons Liquid
- [ ] Documentar aprendizados (blog, GitHub)

**Conectar**:
- [ ] Entrar no Discord Blockstream
- [ ] Seguir builders no Twitter
- [ ] Participar discussions GitHub
- [ ] Apresentar projetos na comunidade

---

**Última Atualização**: 2024-11-14

---

**Comparação Final - Liquid no Ecossistema Bitcoin L2**:

| Use Case | Melhor Escolha | Por Quê |
|----------|----------------|---------|
| Micropagamentos | ⚡ Lightning | Instantâneo, custos mínimos |
| Trading rápido | 🌊 Liquid | Confidential, 2 min final |
| DeFi complexo | 🔶 Rootstock | EVM, ecosystem maduro |
| Stablecoins institucionais | 🌊 Liquid | Privacidade, compliance |
| Smart contracts verificáveis | 🌊 Liquid | Simplicity |
| Máxima compatibilidade Ethereum | 🔶 Rootstock | EVM 100% |
| Streams de pagamento | ⚡ Lightning | Micropagamentos contínuos |
| Securities tokenizadas | 🌊 Liquid | Confidential + compliance |

**Liquid Network** se destaca quando você precisa de **privacidade + velocidade + assets nativos**, sendo a escolha ideal para exchanges, trading desks, emissores de stablecoins e aplicações empresariais que valorizam confidencialidade.

🚀 **Comece hoje**: `docker run -d blockstream/elementsd:latest -chain=liquidtestnet`
