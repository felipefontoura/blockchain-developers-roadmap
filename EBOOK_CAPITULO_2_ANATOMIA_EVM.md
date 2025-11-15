# Capítulo 2: Anatomia da EVM - Como Funciona Por Baixo

> **Para Desenvolvedores Experientes**: Se você já trabalhou com JVM, Python VM ou qualquer máquina virtual, este capítulo vai mapear esses conceitos para o mundo Ethereum. Se você entende como funciona um interpretador ou runtime, está à frente.

---

## Índice
- [2.1 O que é a EVM e Por Que Ela Existe](#21-o-que-é-a-evm-e-por-que-ela-existe)
- [2.2 Arquitetura: Stack-Based vs Register-Based](#22-arquitetura-stack-based-vs-register-based)
- [2.3 Storage, Memory e Stack: O Modelo de Dados](#23-storage-memory-e-stack-o-modelo-de-dados)
- [2.4 Gas: O Sistema de Recursos](#24-gas-o-sistema-de-recursos)
- [2.5 O Ciclo de Vida de uma Transação](#25-o-ciclo-de-vida-de-uma-transação)
- [2.6 Bytecode e Opcodes](#26-bytecode-e-opcodes)
- [2.7 Deep Dive: Como o Compilador Solidity Funciona](#27-deep-dive-como-o-compilador-solidity-funciona)

---

## 2.1 O que é a EVM e Por Que Ela Existe

### Comparação com Sistemas Tradicionais

Se você vem de desenvolvimento tradicional, pense na EVM assim:

| Conceito | Sistema Tradicional | EVM/Blockchain |
|----------|-------------------|----------------|
| **Runtime** | JVM, Node.js, Python Interpreter | Ethereum Virtual Machine |
| **Código** | JAR, .pyc, binário | Bytecode EVM (hex) |
| **Execução** | Servidor centralizado | Milhares de nodes simultaneamente |
| **Estado** | Banco de dados (MySQL, Postgres) | State Tree (Merkle Patricia Trie) |
| **Custo** | CPU time, RAM | Gas (medido em Wei) |
| **Falha** | Exception, Error code | Revert (rollback completo) |

### 📖 Glossário de Termos Web3

**Node (Nó)**
> Um computador que executa o software Ethereum e mantém uma cópia da blockchain. Cada node valida transações e executa smart contracts independentemente. Pense como "servidor na rede P2P".

**State Tree (Merkle Patricia Trie)**
> Estrutura de dados que armazena o estado global da blockchain (saldos, storage de contratos, etc.). É como um "banco de dados distribuído" onde cada alteração gera um novo hash raiz, permitindo verificação criptográfica.
>
> **Analogia**: Imagine o Git - cada commit tem um hash único. Na blockchain, cada bloco tem um state root hash que representa o estado completo naquele momento.

**Wei**
> A menor unidade de Ether (ETH). `1 ETH = 10^18 Wei` (1 quintilhão de wei)
>
> **Analogia**: Como centavos para dólares, mas em escala muito maior:
> - 1 Wei = 0.000000000000000001 ETH
> - 1 Gwei (Gigawei) = 1,000,000,000 Wei = 10^-9 ETH
> - Gas prices são geralmente medidos em Gwei

**Revert**
> Desfaz TODAS as alterações de estado de uma transação quando algo dá errado. Diferente de try/catch tradicional, aqui é tudo-ou-nada (atomic).
>
> **Analogia**: Como ROLLBACK em SQL - se qualquer parte falha, toda a transação é desfeita. Mas o gas já consumido NÃO é devolvido.

### Por Que Uma VM?

**Problema**: Como executar código não-confiável em milhares de computadores diferentes (Linux, Windows, Mac, ARM, x86) de forma:
- ✅ Determinística (mesmo input = mesmo output, sempre)
- ✅ Segura (sem acesso ao sistema operacional)
- ✅ Mensurável (para cobrar pelo processamento)

**Solução**: Uma máquina virtual com:
1. **Ambiente sandbox** completo
2. **Instruções de baixo nível** (opcodes) deterministicas
3. **Medição de recursos** (gas) para cada operação

### 🏗️ Analogia Arquitetural

```
┌─────────────────────────────────────────────────────┐
│                   APLICAÇÃO WEB2                    │
├─────────────────────────────────────────────────────┤
│  Frontend (React) ←→ API (Node.js) ←→ DB (Postgres) │
│       ↓                  ↓                 ↓        │
│   Navegador          Servidor           Servidor    │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                   APLICAÇÃO WEB3                    │
├─────────────────────────────────────────────────────┤
│  Frontend (React) ←→ Smart Contract ←→ State        │
│       ↓                  ↓                 ↓        │
│   Navegador            EVM              Blockchain   │
│                    (replicada           (replicado   │
│                   em 1000s nodes)      em 1000s)     │
└─────────────────────────────────────────────────────┘
```

**Implicação Crítica**: Quando você escreve `balance[user] = 100`, isso é replicado e executado em milhares de computadores. Por isso:
- Operações custam gas ⛽
- Código não pode ser "um pouco errado"
- Bugs são imutáveis (ou muito caros de corrigir)

---

## 2.2 Arquitetura: Stack-Based vs Register-Based

### Quick Refresher

**Register-based (x86, ARM, JVM)**:
```assembly
MOV R1, 5      ; Registrador R1 = 5
MOV R2, 3      ; Registrador R2 = 3
ADD R3, R1, R2 ; R3 = R1 + R2 = 8
```

**Stack-based (EVM, Python VM)**:
```assembly
PUSH 5    ; Stack: [5]
PUSH 3    ; Stack: [5, 3]
ADD       ; Stack: [8]  (pop 5 e 3, push 8)
```

### Por Que a EVM é Stack-Based?

1. **Simplicidade**: Menos estado para gerenciar (sem registradores nomeados)
2. **Bytecode compacto**: Instruções menores = menos dados na blockchain
3. **Fácil de verificar**: Análise estática mais simples

⚠️ **Trade-off**: Pode ser menos eficiente que register-based, mas determinismo e verificabilidade são mais importantes.

---

## 2.3 Storage, Memory e Stack: O Modelo de Dados

Esta é **a parte mais importante** para entender custos e design de smart contracts.

### Os Três Locais de Armazenamento

```
┌──────────────────────────────────────────────────────────┐
│                    SMART CONTRACT                        │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ╔═══════════════╗   ╔════════════╗   ╔══════════════╗  │
│  ║    STORAGE    ║   ║   MEMORY   ║   ║    STACK     ║  │
│  ║               ║   ║            ║   ║              ║  │
│  ║ • Persistente ║   ║ • Temporário  ║ ║ • Temporário ║  │
│  ║ • 2^256 slots ║   ║ • Byte array  ║ ║ • 256-bit    ║  │
│  ║ • CARO $$$    ║   ║ • Barato $    ║ ║ • Barato $   ║  │
│  ║ • 20k gas/    ║   ║ • 3 gas/      ║ ║ • 3 gas/     ║  │
│  ║   write       ║   ║   32 bytes    ║ ║   push       ║  │
│  ║               ║   ║            ║   ║              ║  │
│  ║ mapping(...)  ║   ║ function() ║   ║ Operações    ║  │
│  ║ uint256 x;    ║   ║ {          ║   ║ internas     ║  │
│  ║               ║   ║   bytes    ║   ║              ║  │
│  ║               ║   ║   memory..}║   ║              ║  │
│  ╚═══════════════╝   ╚════════════╝   ╚══════════════╝  │
│       ↓                    ↓                  ↓         │
│  Entre chamadas       Apenas durante      Apenas        │
│  (persiste)           execução           durante        │
│                                          execução       │
└──────────────────────────────────────────────────────────┘
```

### 1. Storage (Persistente)

**Analogia**: Banco de dados permanente (como tabelas SQL)

```solidity
contract Example {
    uint256 public count;  // Slot 0 no storage
    address public owner;  // Slot 1 no storage
    mapping(address => uint256) public balances; // Slot 2+

    // Cada write em storage custa ~20,000 gas!
    function increment() public {
        count++; // CARO: lê storage, incrementa, escreve storage
    }
}
```

**Custos de Gas**:
- **SLOAD** (ler): 2,100 gas (cold) / 100 gas (warm)
- **SSTORE** (escrever): 20,000 gas (novo) / 5,000 gas (update)

💡 **Pro Tip**: Minimize writes no storage. Exemplo:

```solidity
// ❌ INEFICIENTE - 3 writes no storage
function badTransfer(address to, uint amount) public {
    balances[msg.sender] -= amount;  // Write 1
    balances[to] += amount;           // Write 2
    totalTransfers++;                 // Write 3
}

// ✅ OTIMIZADO - use memory para cálculos
function goodTransfer(address to, uint amount) public {
    uint256 senderBalance = balances[msg.sender]; // 1 read
    require(senderBalance >= amount);

    unchecked { // Economiza gas check overflow (se você tem certeza)
        balances[msg.sender] = senderBalance - amount;  // 1 write
        balances[to] += amount;                          // 1 write
    }
    // totalTransfers pode ser calculado off-chain via events!
}
```

### 2. Memory (Temporário)

**Analogia**: Variáveis locais em uma função (como heap em C)

```solidity
function processData(uint256[] calldata ids) public view returns (uint256[] memory) {
    // Array criado em MEMORY (não persiste após a função)
    uint256[] memory results = new uint256[](ids.length);

    for(uint i = 0; i < ids.length; i++) {
        results[i] = ids[i] * 2; // Operações em memory são baratas
    }

    return results; // Memory é copiada para o calldata de retorno
}
```

**Características**:
- ✅ ~100x mais barato que storage
- ✅ Pode alocar dinamicamente
- ❌ Não persiste após a transação
- ❌ Expande linearmente (custo aumenta com uso)

### 3. Stack (Operações Internas)

**Analogia**: CPU registers (mas como stack)

- Profundidade máxima: **1024 itens**
- Cada item: **256 bits (32 bytes)**
- Usado para: operações aritméticas, lógica, jumps

⚠️ **Stack Too Deep Error**:
```solidity
function tooManyVariables() public {
    uint a = 1;
    uint b = 2;
    // ...
    uint z = 26; // Erro! Mais de 16 variáveis locais
}

// Solução: use structs ou reduza variáveis
struct Vars {
    uint a;
    uint b;
    // ...
}
```

### 🔒 Security: Storage Collision

```solidity
// ❌ VULNERÁVEL - storage collision em upgradeable contracts
contract V1 {
    uint256 public value; // Slot 0
}

contract V2 is V1 {
    address public owner; // PERIGO! Também vai pro slot 0 se não herdar corretamente
    uint256 public value; // Agora é slot 1, mas V1 achava que era 0!
}
```

**Solução**: Use padrão de storage gaps ou diamond pattern.

---

## 2.4 Gas: O Sistema de Recursos

### Por Que Gas Existe?

Sem gas, alguém poderia fazer isso:

```solidity
// Ataque de DoS à rede inteira
function attack() public {
    while(true) {
        // Loop infinito
    }
}
```

**Gas resolve**:
1. **Halting Problem**: Previne loops infinitos (gas limit)
2. **Resource Pricing**: Operações caras custam mais
3. **Spam Protection**: Transações custam dinheiro

### Anatomia do Gas

```
┌─────────────────────────────────────────────────────────┐
│                    USUÁRIO PAGA                         │
├─────────────────────────────────────────────────────────┤
│  Gas Usado × Gas Price = Custo em ETH                  │
│                                                          │
│  21,000 × 50 gwei = 0.00105 ETH (~$2 USD)               │
│    ↑         ↑                                           │
│    │         └─ Quanto você está disposto a pagar/gas   │
│    └─────────── Gas consumido pela transação            │
└─────────────────────────────────────────────────────────┘
```

### Custos de Operações Comuns

| Operação | Gas | Analogia |
|----------|-----|----------|
| Adição/Subtração | 3 | CPU cycle |
| Multiplicação | 5 | CPU cycle |
| Divisão | 5 | CPU cycle |
| SLOAD (ler storage) | 2,100 | Ler do disco (frio) |
| SSTORE (escrever) | 20,000 | Escrever no disco |
| LOG (event) | 375 + dados | Write-ahead log |
| CREATE (novo contrato) | 32,000 | Criar tabela DB |
| Transaction base | 21,000 | Overhead de rede |

### Otimização de Gas na Prática

```solidity
contract GasOptimization {

    // ❌ INEFICIENTE: 50,000+ gas
    function badLoop(uint256[] memory data) public {
        for(uint256 i = 0; i < data.length; i++) {
            // Lê data.length toda iteração (memory read)
            storage[i] = data[i]; // Storage write
        }
    }

    // ✅ OTIMIZADO: ~30,000 gas
    function goodLoop(uint256[] memory data) public {
        uint256 length = data.length; // Cache length
        for(uint256 i = 0; i < length;) {
            storage[i] = data[i];
            unchecked { i++; } // Sem overflow check (economiza 20 gas/loop)
        }
    }

    // ✅✅ MUITO OTIMIZADO: Use assembly para casos críticos
    function assemblyLoop(uint256[] memory data) public {
        assembly {
            let length := mload(data)
            let dataPtr := add(data, 0x20)

            for { let i := 0 } lt(i, length) { i := add(i, 1) } {
                let value := mload(add(dataPtr, mul(i, 0x20)))
                sstore(i, value)
            }
        }
    }
}
```

💡 **Pro Tip**: Use [gas-reporter](https://github.com/cgewecke/eth-gas-reporter) em seus testes para medir consumo real.

---

## 2.5 O Ciclo de Vida de uma Transação

### 📖 Termos-Chave de Transações

Antes de entender o fluxo, vamos definir conceitos essenciais:

**Transação (Transaction)**
> Dados assinados criptograficamente que instruem a EVM a executar uma ação (transferir ETH, chamar função de contrato, fazer deploy, etc.).
>
> **Analogia Web2**: Como uma requisição HTTP POST, mas:
> - Assinada com criptografia (não pode ser forjada)
> - Imutável após ser incluída em um bloco
> - Custa dinheiro (gas) para ser processada

**Private Key / Public Key**
> Par de chaves criptográficas (ECDSA - Elliptic Curve Digital Signature Algorithm):
> - **Private Key**: Senha secreta (64 caracteres hex). NUNCA compartilhe!
> - **Public Key**: Derivada da private key, usada para gerar seu endereço
> - **Endereço Ethereum**: Hash da public key (0x123... com 40 caracteres hex)
>
> **Analogia**: Private key = senha do banco, Public key/Address = número da conta

**Nonce**
> Número sequencial de transações enviadas por uma conta (começa em 0).
> Previne "replay attacks" - você não pode enviar a mesma transação duas vezes.
>
> **Analogia**: Como número de cheque - cada transação tem um número único e sequencial.

**Mempool (Memory Pool)**
> "Sala de espera" de transações pendentes. Cada node mantém um mempool com transações válidas aguardando inclusão em bloco.
>
> **Analogia Web2**: Como uma fila de jobs/tasks aguardando processamento. Mas é descentralizada - cada node tem sua própria fila.

**MEV (Maximal Extractable Value)**
> Lucro que mineradores/validadores podem obter reordenando, incluindo ou excluindo transações dentro de um bloco.
>
> **Exemplo**: Ver que alguém vai comprar muito de um token → comprar antes → vender depois (front-running)
>
> **Analogia**: Como um corretor vendo ordens de compra de clientes e negociando primeiro para si mesmo (mas legal em blockchain!).

**Validador/Minerador**
> - **Minerador (PoW)**: Competia resolvendo puzzle matemático para criar blocos (Bitcoin, Ethereum antiga)
> - **Validador (PoS)**: Selecionado aleatoriamente para propor blocos, baseado no stake (ETH apostado)
>
> **Analogia**: Como um notário que valida documentos, mas descentralizado

### Fluxo Completo

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. USUÁRIO ASSINA TRANSAÇÃO                                     │
│    → Metamask cria tx: {to, value, data, gasLimit, gasPrice}    │
│    → Assina com private key (ECDSA)                             │
│    → Gera assinatura: (v, r, s) - prova criptográfica           │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│ 2. BROADCAST PARA MEMPOOL                                        │
│    → Tx vai para mempool de nodes via protocolo P2P             │
│    → Validação básica (assinatura válida, nonce correto,        │
│      balance suficiente para gas)                               │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│ 3. VALIDADOR SELECIONA TX                                        │
│    → Ordena por gas price + priority fee (quem paga mais)       │
│    → Considera MEV (pode reordenar para lucro)                  │
│    → Inclui no próximo bloco (~12 segundos no Ethereum)         │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│ 4. EXECUÇÃO NA EVM                                               │
│    ┌────────────────────────────────────────────────────────┐   │
│    │ a. Carrega bytecode do endereço 'to'                  │   │
│    │ b. Inicializa contexto (msg.sender, msg.value, etc)   │   │
│    │ c. Executa opcode por opcode                          │   │
│    │ d. Consome gas a cada operação                        │   │
│    │ e. Atualiza state tree                                │   │
│    │ f. Emite events (logs)                                │   │
│    │ g. Retorna resultado ou reverte                       │   │
│    └────────────────────────────────────────────────────────┘   │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│ 5. FINALIZAÇÃO                                                   │
│    → Se sucesso: state changes são commitados                   │
│    → Se falha: reverte tudo (mas gas é consumido!)              │
│    → Receipt é criado (logs, status, gasUsed)                   │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│ 6. PROPAGAÇÃO                                                    │
│    → Bloco é propagado para outros nodes                        │
│    → Cada node re-executa as transações (verificação)           │
│    → Após ~15 blocos: considerado "confirmado"                  │
└──────────────────────────────────────────────────────────────────┘
```

### Contexto de Execução (msg.* e tx.*)

### 📖 Variáveis Globais da EVM

**msg.sender**
> Endereço que chamou DIRETAMENTE a função atual. Pode ser uma wallet (EOA) ou outro contrato.
>
> **Analogia Web2**: Como `req.user.id` em uma API - quem está fazendo a requisição agora.

**msg.value**
> Quantidade de Wei (ETH) enviada junto com a chamada da função.
>
> **Analogia**: Como enviar dinheiro junto com um formulário - "aqui está meu pagamento E minha requisição".

**msg.data (Calldata)**
> Bytes brutos enviados na transação. Inclui function selector (4 bytes) + parâmetros encodados.
>
> **Analogia Web2**: Como o body de uma requisição HTTP POST, mas em formato binário compacto.
>
> **Exemplo**: Chamar `transfer(0x123, 100)` gera:
> - `0xa9059cbb` (function selector)
> - `000000000000000000000000123...` (endereço)
> - `0000000000000000000000000064` (100 em hex)

**tx.origin**
> Endereço da conta EOA (Externally Owned Account - wallet com private key) que INICIOU a cadeia de chamadas.
>
> **Diferença crítica de msg.sender**:
> - User → ContractA → ContractB
> - Em ContractB: `msg.sender = ContractA`, `tx.origin = User`
>
> ⚠️ **NUNCA use tx.origin para autenticação!** (vulnerável a phishing)

**Block Variables**
> Informações sobre o bloco atual:
> - `block.number`: Altura do bloco (quantos blocos desde o genesis)
> - `block.timestamp`: Unix timestamp aproximado do bloco
> - `block.coinbase`: Endereço do validador que criou este bloco
>
> ⚠️ **Atenção**: Mineradores/validadores podem manipular timestamp em ~15 segundos

**gasleft()**
> Quanto gas ainda resta para executar a transação.
>
> **Analogia**: Como verificar bateria restante antes de executar uma operação cara.

Quando um smart contract executa, ele tem acesso a:

```solidity
contract Context {
    function showContext() public payable {
        // Quem chamou DIRETAMENTE este contrato
        address sender = msg.sender;

        // Quanto ETH foi enviado (em Wei)
        uint256 value = msg.value;

        // Os dados enviados (calldata) - bytes brutos
        bytes memory data = msg.data;

        // Quanto gas resta
        uint256 gasLeft = gasleft();

        // Quem iniciou a transação (EOA original)
        // ⚠️ Pode ser diferente de msg.sender!
        address origin = tx.origin;

        // Gas price desta tx (quanto paga por gas)
        uint256 gasPrice = tx.gasprice;

        // Informações do bloco
        uint256 blockNumber = block.number;
        uint256 timestamp = block.timestamp;
        address miner = block.coinbase;
    }
}
```

⚠️ **Security: tx.origin vs msg.sender**

```solidity
// ❌ VULNERÁVEL - Phishing attack
contract Vulnerable {
    address owner;

    function withdraw() public {
        require(tx.origin == owner); // NUNCA USE tx.origin para auth!
        payable(owner).transfer(address(this).balance);
    }
}

// Ataque:
// 1. Atacante cria contrato malicioso
// 2. Convence owner a chamar contrato malicioso
// 3. Contrato malicioso chama Vulnerable.withdraw()
// 4. tx.origin ainda é owner, então passa!

// ✅ SEGURO
contract Safe {
    address owner;

    function withdraw() public {
        require(msg.sender == owner); // Use msg.sender!
        payable(owner).transfer(address(this).balance);
    }
}
```

---

## 2.6 Bytecode e Opcodes

### Do Solidity ao Bytecode

```solidity
// Solidity
function add(uint a, uint b) public pure returns (uint) {
    return a + b;
}
```

**Compila para:**

```
BYTECODE (hex):
6080604052348015600f57600080fd5b5060043610...

OPCODES (assembly):
PUSH1 0x80
PUSH1 0x40
MSTORE
CALLVALUE
DUP1
ISZERO
PUSH1 0x0F
JUMPI
...
```

### Opcodes Fundamentais

| Categoria | Opcode | Descrição | Gas |
|-----------|--------|-----------|-----|
| **Stack** | PUSH1-32 | Coloca valor na stack | 3 |
| | POP | Remove topo da stack | 2 |
| | DUP1-16 | Duplica item na stack | 3 |
| **Aritmética** | ADD | Adição | 3 |
| | SUB | Subtração | 3 |
| | MUL | Multiplicação | 5 |
| | DIV | Divisão | 5 |
| **Comparação** | LT | Menor que | 3 |
| | GT | Maior que | 3 |
| | EQ | Igual | 3 |
| **Storage** | SLOAD | Ler storage | 2100 |
| | SSTORE | Escrever storage | 20000 |
| **Memory** | MLOAD | Ler memory | 3 |
| | MSTORE | Escrever memory | 3 |
| **Controle** | JUMP | Pular para PC | 8 |
| | JUMPI | Pular se condição | 10 |
| | REVERT | Reverter execução | 0 |
| **Contexto** | CALLER | msg.sender | 2 |
| | CALLVALUE | msg.value | 2 |
| | BALANCE | Balance de endereço | 400 |

### Exemplo: Analisando Bytecode

```solidity
contract Simple {
    uint256 public number;

    function setNumber(uint256 _number) public {
        number = _number;
    }
}
```

**Bytecode deployment**:
```
608060405234801561001057600080fd5b50610150...
```

**Como ler** (simplificado):
```
60 80       → PUSH1 0x80 (free memory pointer)
60 40       → PUSH1 0x40
52          → MSTORE (armazena 0x80 na posição 0x40)
34          → CALLVALUE (pega msg.value)
80          → DUP1
15          → ISZERO
60 0f       → PUSH1 0x0f
57          → JUMPI (pula se msg.value == 0)
...
```

💡 **Pro Tip**: Use [evm.codes](https://www.evm.codes/) para explorar opcodes interativamente.

---

## 2.7 Deep Dive: Como o Compilador Solidity Funciona

### Pipeline de Compilação

```
┌─────────────────────────────────────────────────────────────┐
│  SOLIDITY SOURCE CODE (.sol)                                │
│  contract Token { ... }                                     │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  PARSER                                                     │
│  → Lexical analysis (tokens)                                │
│  → Syntax analysis (AST - Abstract Syntax Tree)             │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  SEMANTIC ANALYSIS                                          │
│  → Type checking                                            │
│  → Name resolution                                          │
│  → Control flow analysis                                    │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  INTERMEDIATE REPRESENTATION (IR)                           │
│  → Yul (intermediate language)                              │
│  → Optimizations (constant folding, dead code elimination)  │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  CODE GENERATION                                            │
│  → EVM Bytecode                                             │
│  → Function selectors (4-byte)                              │
│  → ABI (Application Binary Interface)                       │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  OUTPUT                                                     │
│  • Bytecode (para deploy)                                   │
│  • Runtime bytecode (código final no blockchain)            │
│  • ABI JSON (para front-end interagir)                      │
│  • Metadata (verificação, source maps)                      │
└─────────────────────────────────────────────────────────────┘
```

### Function Selectors (Como a EVM Sabe Qual Função Chamar)

### 📖 ABI e Function Selectors

**ABI (Application Binary Interface)**
> JSON que descreve como interagir com um smart contract: nomes de funções, tipos de parâmetros, eventos, etc.
>
> **Analogia Web2**: Como um arquivo de especificação OpenAPI/Swagger, mas para smart contracts.
>
> **Exemplo de ABI**:
> ```json
> [{
>   "name": "transfer",
>   "type": "function",
>   "inputs": [
>     {"name": "to", "type": "address"},
>     {"name": "amount", "type": "uint256"}
>   ],
>   "outputs": [{"type": "bool"}]
> }]
> ```

**Function Selector**
> Hash de 4 bytes (primeiros 4 bytes do keccak256) usado para identificar qual função chamar.
>
> **Por que existe**: O bytecode do contrato não tem "nomes" de funções - apenas endereços de memória. O selector mapeia nome → endereço.
>
> **Analogia**: Como um "índice" ou "roteador" - similar a como HTTP usa GET/POST + path para rotear requisições.

**Keccak256**
> Função de hash criptográfico usada pela Ethereum (variante do SHA-3).
>
> **Diferença importante**: `keccak256 ≠ SHA3-256` oficial (Ethereum usa a versão pré-padronização)

```solidity
contract Example {
    function transfer(address to, uint256 amount) public {
        // ...
    }

    function balanceOf(address account) public view returns (uint256) {
        // ...
    }
}
```

**Como funciona o dispatch de funções**:
1. Front-end chama `transfer(0x123..., 100)`
2. Calcula function selector: `keccak256("transfer(address,uint256)")`
3. Pega primeiros 4 bytes: `0xa9059cbb` ← **Function Selector**
4. Encoda parâmetros usando ABI encoding:
   - `000000000000000000000000123...` (endereço, 32 bytes)
   - `0000000000000000000000000064` (100 em hex, 32 bytes)
5. Calldata final: `0xa9059cbb` + parâmetros encodados
6. Contrato lê primeiros 4 bytes, compara com selectors conhecidos, pula para código correto

**No bytecode do contrato**:
```assembly
CALLDATA_LOAD    ; Carrega os primeiros 4 bytes
PUSH 0xa9059cbb  ; Function selector de transfer
EQ               ; Compara
JUMPI            ; Se igual, pula para código de transfer
PUSH 0x70a08231  ; Senão, tenta balanceOf
EQ
JUMPI
REVERT           ; Função não encontrada
```

### 💡 Pro Tip: Otimizador Solidity

```bash
# Compilar com otimização
solc --optimize --optimize-runs 200 contract.sol

# Mais runs = mais caro deploy, mais barato execução
# Menos runs = mais barato deploy, mais caro execução

# Para contratos muito usados (tipo USDT):
--optimize-runs 1000000

# Para contratos de deploy único:
--optimize-runs 1
```

---

## 🔒 Security Checklist: EVM Internals

Antes de fazer deploy, considere:

- [ ] **Integer Overflow/Underflow**: Use Solidity 0.8+ (checks automáticos) ou SafeMath
- [ ] **Reentrancy**: Checks-Effects-Interactions pattern
- [ ] **Gas Limit**: Funções não devem depender de loops ilimitados
- [ ] **tx.origin**: Nunca use para autenticação
- [ ] **Storage Collision**: Cuidado com upgradeable contracts
- [ ] **Function Selector Collision**: Raro, mas possível
- [ ] **Delegatecall**: Contexto de storage é do caller!
- [ ] **Randomness**: block.timestamp e blockhash são manipuláveis

---

## 📖 Glossário Consolidado Web3/Blockchain

Esta seção consolida TODOS os termos específicos de blockchain mencionados no capítulo:

### Conceitos Fundamentais

**EOA (Externally Owned Account)**
> Conta controlada por private key (sua wallet). Diferente de Contract Account.
> - Pode iniciar transações
> - Não tem código
> - Exemplos: MetaMask, Ledger, carteiras mobile

**Contract Account**
> Conta que contém código (smart contract). Não tem private key.
> - Não pode iniciar transações sozinha
> - Executada quando recebe transação
> - Tem storage próprio

**Testnet vs Mainnet**
> - **Mainnet**: Blockchain "real" com ETH real (custa dinheiro de verdade)
> - **Testnet**: Blockchains de teste (Goerli, Sepolia) com ETH de teste (grátis)
>
> **Analogia**: Mainnet = produção, Testnet = ambiente de staging/dev

**Confirmações de Bloco**
> Quantos blocos foram minerados DEPOIS do bloco contendo sua transação.
> - 1 confirmação = incluída em bloco, mas pode ser revertida
> - 12-15 confirmações = considerado seguro (probabilidade de reverter é ~0)
>
> **Analogia**: Como esperar compensação de um cheque - quanto mais tempo passa, mais seguro que não vai voltar.

### Termos de Smart Contracts

**Delegatecall**
> Chama código de outro contrato mas usa storage/contexto do contrato atual.
>
> **Analogia**: Como importar uma biblioteca mas ela manipula SEUS dados, não os dela.
>
> ⚠️ **Perigo**: Se não for cuidadoso, contrato externo pode sobrescrever seu storage!

**Fallback Function**
> Função executada quando:
> - Chamada não corresponde a nenhum function selector
> - ETH é enviado sem calldata
>
> ```solidity
> fallback() external payable {
>     // Código executado quando função não é encontrada
> }
>
> receive() external payable {
>     // Executado quando só ETH é enviado (sem data)
> }
> ```

**Events / Logs**
> Forma de emitir dados da blockchain (não ficam em storage, ficam em logs da transação).
>
> ```solidity
> event Transfer(address indexed from, address indexed to, uint256 amount);
> emit Transfer(msg.sender, recipient, 100);
> ```
>
> **Analogia Web2**: Como logs do servidor ou webhooks - registram que algo aconteceu.
> **Vantagem**: Muito mais barato que storage (~375 gas vs 20,000 gas)

**View vs Pure vs Payable**
> Modificadores que indicam comportamento da função:
> - **view**: Lê state mas não modifica (read-only, sem custo de gas se chamada diretamente)
> - **pure**: Não lê nem modifica state (funções matemáticas puras)
> - **payable**: Pode receber ETH
> - (sem modificador): Modifica state, não aceita ETH

### Termos Técnicos de Execução

**OPCODE**
> Instruções de baixo nível da EVM (assembly). Cada uma tem custo de gas específico.
> - Exemplo: `ADD` (adição), `SSTORE` (escrever storage), `CALL` (chamar outro contrato)

**PC (Program Counter)**
> Ponteiro que indica qual instrução (opcode) está sendo executada agora.
>
> **Analogia**: Como um cursor percorrendo linha por linha do código.

**Call Stack**
> Pilha de chamadas de contratos. Máximo de profundidade: 1024.
>
> **Analogia**: Como call stack tradicional, mas com limite explícito para prevenir ataques de recursão infinita.

**Yul**
> Linguagem intermediária entre Solidity e EVM bytecode.
> Permite escrever código assembly mais legível que opcodes brutos.
>
> ```solidity
> assembly {
>     let x := mload(0x40)  // Yul (mais legível)
>     // vs
>     PUSH1 0x40
>     MLOAD              // Opcodes brutos
> }
> ```

### Conceitos Avançados

**Reentrancy**
> Vulnerabilidade onde contrato externo chama de volta seu contrato antes da primeira chamada terminar.
>
> **Exemplo famoso**: The DAO hack (2016, $60 milhões roubados)
>
> **Analogia**: Como recursão não intencional onde atacante "interrompe" sua função e a chama de novo.

**Flash Loan**
> Empréstimo que deve ser pago de volta na MESMA transação.
> - Se não pagar, toda transação reverte (atomicidade)
> - Sem colateral necessário
> - Usado para arbitragem, liquidações, etc.

**Slippage**
> Diferença entre preço esperado e preço executado (comum em DEXs).
>
> **Analogia**: Como comprar ação - entre você ver o preço e executar a ordem, pode ter mudado.

**Oracle**
> Serviço que traz dados externos (preços, clima, resultados esportivos) para dentro da blockchain.
> - Chainlink é o mais popular
> - Necessário porque EVM não pode acessar internet diretamente

**L2 (Layer 2)**
> Soluções de escalabilidade que processam transações fora da mainnet (L1) mas herdam sua segurança.
> - Exemplos: Arbitrum, Optimism, Polygon zkEVM
> - Mais rápido e barato que L1

**EIP (Ethereum Improvement Proposal)**
> Propostas de mudanças no protocolo Ethereum.
> - EIP-20 → Padrão de tokens (ERC-20)
> - EIP-721 → Padrão de NFTs (ERC-721)
> - EIP-1559 → Mudança no modelo de gas fees

### Unidades de Medida

**Unidades de ETH**:
```
1 Wei        = 1 Wei
1 Kwei       = 10^3 Wei    (mil)
1 Mwei       = 10^6 Wei    (milhão)
1 Gwei       = 10^9 Wei    (bilhão) ← Gas prices
1 Szabo      = 10^12 Wei
1 Finney     = 10^15 Wei
1 Ether (ETH)= 10^18 Wei
```

**Tamanhos**:
- 1 byte = 8 bits
- 1 word (EVM) = 32 bytes = 256 bits
- Endereço Ethereum = 20 bytes (40 caracteres hex com 0x)
- Hash (keccak256) = 32 bytes (64 caracteres hex)

---

## 📝 Exercícios Práticos

### Exercício 1: Gas Golf

Otimize este contrato para usar o mínimo de gas possível:

```solidity
contract GasGolf {
    uint256[] public values;

    // Objetivo: armazenar números de 1 a 100
    function populate() public {
        for(uint256 i = 1; i <= 100; i++) {
            values.push(i);
        }
    }

    // Objetivo: somar todos os valores
    function sum() public view returns (uint256) {
        uint256 total = 0;
        for(uint256 i = 0; i < values.length; i++) {
            total += values[i];
        }
        return total;
    }
}
```

**Desafio**: Reduza o gas usado em pelo menos 50%.

<details>
<summary>💡 Dica</summary>

- Use `unchecked` para economia
- Cache `values.length`
- Consider usar fórmula matemática ao invés de loop no `sum()`
- Será que precisa mesmo armazenar em storage?
</details>

### Exercício 2: Decode Bytecode

Dado este bytecode:
```
6080604052348015600f57600080fd5b5060043610...
```

Use [evm.codes/playground](https://www.evm.codes/playground) para:
1. Identificar o padrão de inicialização
2. Encontrar os function selectors
3. Mapear para funções conhecidas

### Exercício 3: Storage Layout

```solidity
contract StorageLayout {
    uint128 public a; // Slot ?
    uint128 public b; // Slot ?
    uint256 public c; // Slot ?
    bool public d;    // Slot ?
    bool public e;    // Slot ?
}
```

**Questões**:
1. Em quais slots cada variável fica?
2. Como o compilador faz packing?
3. Quanto gas economiza fazer packing?

<details>
<summary>💡 Resposta</summary>

- Slot 0: `a` e `b` (ambos uint128 = 256 bits total)
- Slot 1: `c` (uint256 ocupa slot inteiro)
- Slot 2: `d` e `e` (ambos bool, empacotados)

Packing economiza ~15,000 gas por SSTORE evitado!
</details>

---

## 📚 Recursos Adicionais

### Documentação Essencial
1. **[Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)** - Spec formal da EVM (denso, mas completo)
2. **[evm.codes](https://www.evm.codes/)** - Referência interativa de opcodes
3. **[Solidity Docs - Internals](https://docs.soliditylang.org/en/latest/internals/)** - Como o compilador funciona

### Ferramentas
- **[Remix Debugger](https://remix.ethereum.org/)** - Debug step-by-step
- **[Tenderly](https://tenderly.co/)** - Simular e debugar transações
- **[Foundry's `forge inspect`](https://book.getfoundry.sh/)** - Analisar bytecode, storage layout, gas

### Deep Dives
- **[The EVM Handbook](https://noxx3xxon.notion.site/noxx3xxon/The-EVM-Handbook-bb38e175cc404111a391907c4975426d)** - Guia avançadíssimo
- **[EVM From Scratch](https://www.youtube.com/watch?v=RxL_1AfV7N4)** - Construir uma EVM em Go

---

## 🎯 Próximos Passos

Agora que você entende **como** a EVM funciona, você está pronto para:

→ **Capítulo 3**: Solidity - A Linguagem e Suas Peculiaridades
- Type system
- Design patterns
- Common pitfalls

→ **Capítulo 8**: Security - Top 10 Vulnerabilidades
- Com o conhecimento de EVM internals, você vai entender o **por quê** de cada ataque

---

## 💭 Reflexão Final

**Por que este capítulo importa?**

Muitos desenvolvedores escrevem Solidity sem entender a EVM. Isso funciona... até que:
- ❌ Seu contrato custa $100 de gas por transação
- ❌ Você cria uma vulnerabilidade sutil
- ❌ Um bug custa milhões (vide: The DAO, Parity Wallet)

Entender a EVM é como entender como memória funciona em C, ou como o garbage collector funciona em Java. **Você não precisa pensar nisso 100% do tempo, mas precisa saber quando precisa pensar nisso.**

---

**Autor**: Baseado no material do ITA Blockchain Club + experiência de desenvolvimento
**Última Atualização**: 2025
**Feedback**: [Abra uma issue no GitHub](#)

---

**🚀 Pronto para o próximo capítulo?** O conhecimento de EVM internals vai fazer você escrever Solidity melhor, mais seguro e mais barato.
