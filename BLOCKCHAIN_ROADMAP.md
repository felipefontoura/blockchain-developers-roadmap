# Roadmap de Desenvolvimento Blockchain

## ITA Blockchain Club

---

## Introdução

> **Aviso Importante**: Blockchain é absurdamente complexo e extenso. Não se assuste se não aprender no começo - demora bastante e tem muita coisa para aprender.

### Expectativas Realistas

- **Tempo de estudo**: Mínimo de **10 horas por semana**
- **Duração mínima**: **6 meses de estudo intensivo**
- **Mentalidade**: "Temos o hábito de superestimar o que conseguimos fazer em um mês e subestimar o que conseguimos fazer em um ano"

### Oportunidades

- Participação em **Hackathons** (altamente recomendado)
- Desenvolvimento de software rápido
- Prêmios e networking
- Viagens internacionais pagas
- Mercado em alta demanda

---

## Legenda da Árvore de Tecnologia

### Tipos de Conteúdo

- 🔵 **Círculo**: Conteúdo de aprendizado
- 🟦 **Quadrado**: Curso ou tecnologia
- 🔶 **Losango**: Projeto essencial

### Níveis de Dificuldade dos Projetos

- 🟢 **Verde**: Fácil
- 🟠 **Laranja**: Médio
- 🔴 **Vermelho**: Difícil
- 🟣 **Roxo**: Profissional (projeto robusto e integrado)

> **Meta**: Conseguir fazer projetos fáceis e médios é ótimo. Difícil é interessante. Profissional ninguém vai exigir, mas é um diferencial.

---

## 1. Treinamento Zero - Fundamentos Blockchain

### Conceitos Essenciais

#### 1.1 Fundamentos Básicos

- O que é blockchain
- Por que ela existe
- Variantes de blockchain
- Protocolos de consenso
- Mecanismos de consenso

#### 1.2 Proof of Work vs Proof of Stake

- **Bitcoin**: Proof of Work
  - O que é PoW
  - Como funciona
- **Ethereum**: Proof of Stake (após The Merge)
  - O que é PoS
  - O que foi The Merge
  - Diferenças do Bitcoin

#### 1.3 Conceitos Avançados Iniciais

- **Smart Contracts** (Contratos Inteligentes)
- **DeFi** (Decentralized Finance)
- **NFTs** e sua importância
- **Zero Knowledge Proofs** (básico)
- **Optimistic Rollups**

#### 1.4 Habilidades Práticas Essenciais

- ✅ Operar com **MetaMask**
- ✅ Criar conta na **Binance** (para aprendizado)
- ✅ Entender transações e endereços de contrato
- ✅ **Guardar private keys em locais físicos** (segurança)
- ✅ Praticar com redes de teste (testnet)

### Recursos

- Videoaulas disponíveis online
- Slides do ITA Blockchain Club
- Material atualizado regularmente

---

## 2. Áreas de Atuação em Blockchain

### 2.1 Tokenomics e Economia Cripto

Para quem não curte tanto desenvolvimento, mas gosta de economia e finanças:

- **Exchanges Centralizadas**
- **DEXs** (Exchanges Descentralizadas)
- **Gamify**
- **ICOs** (Initial Coin Offerings)
- **Análise Técnica de Trade**
- **Crypto Research**
  - Volatilidades
  - Nuvens de Ishimoku
  - Liquidity Pools

### 2.2 Desenvolvimento On-Chain vs Off-Chain

#### Off-Chain (Chain Development)

Desenvolver redes blockchain do zero.

**Pré-requisitos**:

- Conhecimento de estruturas de dados (disciplina 6.11)
- Listas lineares
- Algoritmos
- Eficiência (Big O)

**Tecnologias**:

- **Cosmos SDK**: Blockchains Proof of Stake
- **Hyperledger Besu**: Proof of Authority
- Outras SDKs especializadas

> **Nota**: Não é o foco principal do clube, mas é uma área importante.

#### On-Chain (Foco do ITA Blockchain Club)

Desenvolver aplicações na rede blockchain existente.

---

## 3. Por que Ethereum?

### Vantagens da Ethereum

1. **Blockchain Programável**
   - Não serve apenas como registro de transações
   - Permite executar código (smart contracts)
   - Ethereum Virtual Machine (EVM)

2. **Segunda Maior Criptomoeda** do mundo

3. **Compatibilidade EVM**
   - Código funciona em múltiplas blockchains:
     - Binance (BNB Chain)
     - Polygon
     - Avalanche
     - E muitas outras

4. **Integração Web2 + Web3**
   - Conecta aplicações tradicionais com blockchain
   - Melhor ecossistema para iniciantes

---

## 4. Caminho de Desenvolvimento

Você pode começar por dois caminhos principais:

```
Treinamento Zero
      ↓
   ┌──┴──┐
   ↓     ↓
Front-End  Smart Contracts
   ↓     ↓
   └──┬──┘
      ↓
  Integração
      ↓
   Projetos
```

---

## 5. Front-End Development

### 5.1 Front-End 1 - Fundamentos

#### Tecnologias Base

1. **HTML** (Hypertext Markup Language)
   - Esqueleto do website
   - Aprenda o básico

2. **CSS** (Cascade Style Sheets)
   - Estilização básica
   - Organização visual
   - Cores e layout

3. **JavaScript** ⚠️ **IMPORTANTE**
   - Linguagem de programação para web
   - Torna o site dinâmico
   - Manipulação de elementos
   - **NÃO aprenda nas coxas** - estude direitamente!

4. **React** (Introdução)
   - Library JavaScript
   - Boilerplate code (código pronto)
   - Componentes reutilizáveis
   - Essencial para desenvolvimento moderno

#### Recursos de Aprendizado

- **FreeCodeCamp**
- **Curso do Senda** (front-end)

#### 🟢 Projeto 1: Portfólio Básico

**Objetivo**: Criar um portfólio pessoal simples

- Explicar quem você é
- Suas habilidades
- Usar HTML, CSS, JavaScript e React
- Deve ser básico mas funcional

---

### 5.2 Git - Versionamento de Código

#### Por que Git é Essencial?

**Problema Comum**:

```
codigo_v1.py
codigo_v2.py
codigo_versao_que_deu_certo.py
codigo_versao_final.py
codigo_versao_final_final.py
codigo_versao_final_AGORA_VAI.py
```

**Solução com Git**:

- Sistema de versionamento profissional
- Salva "snapshots" do código
- Fácil reverter para versões anteriores
- Trabalho em equipe facilitado

#### Git vs GitHub

- **Git**: Software local de versionamento
- **GitHub**: Plataforma online para hospedar código
  - Rede social de programadores
  - Colaboração
  - Portfolio de projetos

#### Comandos Básicos Essenciais

```bash
git status      # Ver status dos arquivos
git add .       # Adicionar arquivos
git commit      # Salvar versão
git push        # Enviar para GitHub
git pull        # Baixar do GitHub
```

---

### 5.3 Front-End 2 - Profissional

#### Stack Recomendado (ITA Blockchain Club)

**Tecnologias**:

1. **React** (aprofundamento)
2. **Tailwind CSS**
   - Library CSS moderna
   - Componentes prontos
   - Sites mais bonitos facilmente
3. **TypeScript**
   - JavaScript com tipagem
   - Menos erros
   - Código mais robusto
   - Criado pela Microsoft
4. **Vite.js**
   - Build tool moderno
   - Desenvolvimento rápido
5. **React Native** (opcional)
   - Criar aplicativos mobile
   - Mesmo código, múltiplas plataformas
   - Não precisa aprender Kotlin, Swift, etc.

#### Integração Blockchain

1. **Ethers.js** ⚠️ **CRÍTICO**
   - Conecta smart contracts ao front-end
   - **Parte onde mais dá problema**
   - **Seja mestre nisso!**

2. **APIs** (Application Programming Interface)
   - Comunicação entre aplicações
   - Integração com serviços externos
   - Essencial para Web3

#### Stack Alternativo: MERN

- **M**ongoDB (banco de dados)
- **E**xpress (framework)
- **R**eact (front-end)
- **N**ode.js (back-end)

#### 🟢 Projeto 2: Portfólio 2

**Objetivo**: Portfólio melhorado

- Mais funcionalidades
- Design profissional
- Interação com blockchain (opcional)

#### 🟠 Projeto 3: Uniswap Clone (OPCIONAL)

**Objetivo**: Clonar interface da Uniswap

- Praticar todos os comandos
- Não precisa criar design do zero
- Foca na implementação técnica
- Excelente para aprender

**Por que clonar sites é bom?**

- Não precisa pensar em design
- Foca em implementação
- Exercita JavaScript
- Aprende boas práticas

---

### 5.4 Front-End 3 - Avançado (Opcional)

> **Nota**: Apenas se você realmente gostar muito de front-end

#### Tópicos Avançados

- **SASS** (CSS avançado)
- **Design Responsivo**
- **UI/UX Design**
- **Figma** (ferramenta de design)
- **JavaScript Assíncrono**
- **Integração Web3 Avançada**
- **Git Avançado**
  - Continuous Integration/Continuous Deployment (CI/CD)
  - Cherry picking
  - Submodules

---

## 6. Smart Contracts Development

> **Recomendação**: Melhor caminho para iniciantes
>
> - Menos decoreba que front-end
> - Aprende uma linguagem e desenvolve
> - Mais focado em lógica

### 6.1 Smart Contracts 1 - Fundamentos

#### Conceitos Essenciais

1. **Solidity**
   - Linguagem principal para Ethereum
   - Sintaxe similar a JavaScript/C++
   - Específica para blockchain

2. **Remix IDE**
   - Ambiente de desenvolvimento online
   - Ideal para começar
   - Testes rápidos

#### Tópicos de Aprendizado

- Operações com balance
- Libraries
- Fallback and Receive functions
- **Chainlink Price Feed**
- Interação com interfaces
- Deploy de contratos

#### 🟢 Projeto 4: FundMe

**Objetivo**: Contrato de financiamento coletivo

**Funcionalidades**:

- Aceitar depósitos em ETH
- Conversão ETH para USD (Chainlink)
- Apenas owner pode sacar
- **Modifiers** para segurança
  - Prevenir ataques (re-entrance)
  - Controle de acesso

**Aprendizados**:

- Carteira Ethereum programável
- Segurança básica
- Oráculos (Chainlink)

---

### 6.2 Smart Contracts 2 - Frameworks Profissionais

#### Frameworks de Desenvolvimento

Sair do Remix e ir para ambiente profissional.

##### Opção 1: Hardhat (Mais Popular)

- Desenvolvido em JavaScript
- Muito modularizável
- Mais usado no mercado
- Comunidade grande

##### Opção 2: Foundry (Recomendado Atual)

- 100% em Solidity
- Muito rápido
- Testes robustos
- Tendência atual

##### Opção 3: Brownie

- Desenvolvido em Python
- Bom para quem vem de Python
- Menos usado atualmente

> **Recomendação**: Hardhat ou Foundry
>
> - Se gosta de JavaScript → Hardhat
> - Se prefere Solidity puro → Foundry

#### Ferramentas Adicionais

1. **Ganache**
   - Blockchain local para testes
   - Rápido e gratuito
   - Desenvolvimento ágil

2. **Ethers.js**
   - Integração smart contract ↔ front-end
   - Essencial para full-stack

3. **JavaScript** (se escolher Hardhat)
   - Necessário aprender básico

4. **Integração Web3 Básica**
   - Conectar MetaMask
   - Funções simples
   - Transações básicas

#### Conceitos Avançados

1. **Chainlink VRF** (Verifiable Random Function)
   - Números randômicos na blockchain
   - Blockchain é determinística
   - VRF injeta randomicidade externa
   - Difícil mas importante

2. **Chainlink Keepers**
   - Automação temporal
   - "Trazer tempo para blockchain"
   - Funções agendadas

3. **Verificação de Contratos**
   - Publicar código verificado
   - Transparência
   - Confiança do usuário

#### 🟠 Projeto 5: FundMe 2

**Objetivo**: FundMe com integração front-end

**Requisitos**:

- Contrato robusto
- Front-end integrado (pode ser básico)
- Foco na integração
- **Parte mais importante**: fazer smart contract e front-end conversarem

> **Dica**: O front-end pode ser tosco (botão básico em HTML), o importante é a integração!

#### 🟠 Projeto 6: Raffle.sol (Loteria Descentralizada)

**Objetivo**: Sistema de loteria completo

**Funcionalidades**:

- Múltiplas carteiras podem participar
- Depósito de fundos
- Sorteio randômico (Chainlink VRF)
- Distribuição automática do prêmio
- Integração com front-end (opcional)

**Aprendizados**:

- Randomicidade na blockchain
- Gerenciamento de fundos
- Lógica complexa
- Projeto completo e robusto

---

### 6.3 Smart Contracts 3 - Aplicações DeFi e NFT

> **Aqui você vai brilhar!** 🌟

#### 1. Tokens ERC-20

**Criar sua própria criptomoeda**

- Padrão ERC-20
- Supply, decimals, transfers
- Aprovações e allowances

#### 2. NFTs (ERC-721)

**Criar coleções de NFTs**

- Padrão ERC-721
- Mintagem
- Metadados
- Ownership

#### 3. NFT Factory

**Sistema automatizado de criação de NFTs**

- Geração em massa
- Randomicidade garantida
- Unicidade garantida

#### 4. IPFS (InterPlanetary File System)

**Armazenamento descentralizado**

- Onde as imagens do NFT ficam
- Blockchain armazena apenas:
  - Token ID
  - Token URI (link IPFS)
- Imagem fica no IPFS

#### 5. Aave Protocol

**Empréstimos descentralizados**

- Protocolo de lending
- **Flash Loans** ⚡
  - Empréstimos instantâneos
  - Revolucionário
  - Sem colateral (se devolvido na mesma transação)

#### 6. DAO (Decentralized Autonomous Organization)

**Governança on-chain**

- Votação descentralizada
- Propostas
- Execução automática
- Token de governança

#### 7. Otimização de Gas (Opcional)

**Para produção profissional**

- Reduzir custos de transação
- Importante para aplicações reais
- Não essencial para hackathons

---

### 6.4 Projetos Smart Contracts 3

#### 🟢 Projeto 7: Token Próprio

**Objetivo**: Lançar sua criptomoeda

- Criar token ERC-20
- Initial Coin Offering (ICO)
- Na testnet (dinheiro de teste)

#### 🟠 Projeto 8: NFT Minter

**Objetivo**: Sistema de mintagem de NFTs

- Randomicidade garantida
- Unicidade garantida
- Geração de Token URI
- Integração com IPFS

> **Nota**: NFT sem randomicidade/unicidade é fácil. Com essas garantias é bem difícil!

#### 🟠 Projeto 9: Lend and Borrow

**Objetivo**: Sistema de empréstimos

- Emprestar crypto
- Tomar empréstimo
- Juros
- Colateral
- Usar testnet (sem dinheiro real)

#### 🔴 Projeto 10: NFT Marketplace Completo (OPCIONAL MAS FAÇA!)

**Objetivo**: Marketplace full-stack de NFTs

**Por que fazer?**

- Projeto Flagship (bandeira)
- Portfolio impressionante
- Mostra em entrevistas
- Aplica TUDO que aprendeu

**Características**:

- Listagem de NFTs
- Compra e venda
- Leilões
- Ofertas
- Front-end completo
- Integração total

**Dificuldade**:

- Projeto longo
- Complexo
- Mas extremamente valioso

> **"Isso aqui é aquilo que a gente chama de projeto bandeira"**

---

### 6.5 Smart Contracts 4 - Profissional/Especialização

> ⚠️ **Não é útil para Hackathon, mas é útil para carreira**

#### Tópicos Profissionais

1. **Security (Segurança)**
   - Auditorias de contratos
   - Prevenção de exploits
   - Best practices
   - **Pode ganhar dinheiro fazendo auditorias**

2. **Flash Loans** ⚡
   - Útil para hackathons!
   - Arbitragem
   - Estratégias DeFi avançadas

3. **Arbitragem**
   - Trading automatizado
   - Diferenças de preço entre DEXs
   - Pode gerar renda

4. **MEV** (Maximal Extractable Value)
   - **Perigoso!** Cuidado
   - Entender é importante
   - Não recomendado mexer ativamente

5. **Bridges** (Pontes entre Blockchains)
   - Útil para hackathons
   - Transferir ativos entre chains
   - Exemplo: WBTC (Wrapped Bitcoin)

6. **Outro Framework**
   - Se aprendeu Foundry, aprenda Hardhat
   - Se aprendeu Hardhat, aprenda Foundry
   - Empresas usam diferentes ferramentas
   - Importante ser versátil

#### Exemplo Real
>
> Vaga de estágio oferecida exigia Hardhat, mas membros sabiam Brownie ou Foundry. Importante saber múltiplas ferramentas!

---

## 7. Integração: A Parte Mais Importante

### O que Ganha Hackathons?

```
Front-end Bonito + Smart Contracts Funcionando + Integração Perfeita = 🏆
```

### Time Ideal para Hackathon

1. **Especialista em Front-end**
   - UI/UX profissional
   - Interfaces bonitas

2. **Especialista em Smart Contracts**
   - Lógica robusta
   - Segurança

3. **Especialista em Integração**
   - Conecta front + blockchain
   - Ethers.js
   - APIs
   - **A parte mais difícil!**

### Dica Crucial

> "Front-end é o que dá as cerejas do bolo que diferencia entre você tirar terceiro, segundo até primeiro lugar e não ganhar nada."

> "Essa parte aqui [integração] é fundamental. É a parte onde mais dá problema."

---

## 8. Git Intermediário

### Quando Aprender

- Depois de projetos iniciais
- Antes de trabalho em equipe
- Essencial para hackathons

### Conceitos Importantes

- Branches
- Merge conflicts
- Pull requests
- Colaboração em equipe

---

## 9. Back-End (Opcional)

> Blockchain já é back-end, mas às vezes precisa de back-end tradicional

### Tecnologias

#### Banco de Dados

- **Firebase** (mais fácil)
- **MySQL** (relacional)
- **MongoDB** (não-relacional)

#### Frameworks

- **Node.js**
- **Next.js**
- **Express**

#### Outros

- **AWS** (cloud)
- **Stripe** (pagamentos)
- **APIs** em geral

---

## 10. Outras Blockchains e Linguagens

### Ethereum não é a única

#### Solana

- **Linguagens**: C, C++, Rust
- Muito rápida
- Diferente de Solidity

#### Starknet

- **Linguagem**: Cairo
- "Super nojenta" (difícil)
- Zero-knowledge proofs

#### Polkadot

- Ecossistema próprio
- Parachains

#### Cardano

- Foco em academia
- Haskell

### Importante Saber

- Front-end **não recomeça do zero**
- Smart contracts: **maioria recomeça**
- Conceitos são similares
- Sintaxe diferente

---

## 11. Mercado e Oportunidades

### Demanda de Desenvolvedores Blockchain

> "Developer de blockchain está super em falta. Sempre teve, e agora muito mais porque a demanda está crescendo."

### Brasil

- Empresas querendo blockchain
- Tokens de crédito/dívida
- **Real Digital (Drex)**
- Governo financiando hackathons

### Internacional

- Viagens pagas para hackathons
- Membros foram para:
  - 🇵🇹 Portugal
  - 🇭🇰 Hong Kong
  - 🇳🇱 Holanda
  - 🇫🇷 Paris
  - 🇨🇴 Bogotá
  - 🇺🇸 ETH Denver (com vitória!)

### Ganhos

1. **Prêmios de Hackathon**
   - Valores significativos
   - Frequentes

2. **Salários**
   - Alta remuneração
   - Mercado em crescimento

3. **Empreendedorismo**
   - Muitas oportunidades
   - Startups blockchain

---

## 12. Recursos e Referências

### Materiais do ITA Blockchain Club

- Videoaulas
- Slides atualizados
- Playlist completa
- Links disponibilizados

### Cursos Recomendados

- **FreeCodeCamp**
- **Curso do Senda** (front-end)
- Documentação oficial das tecnologias

---

## 13. Dicas Finais e Mentalidade

### Gestão de Expectativas

1. **Não aprenda em uma semana ou um mês**
   - Mínimo 6 meses
   - Dedicação intensa necessária

2. **10 horas por semana**
   - Estudando
   - Desenvolvendo
   - Ou ambos

3. **Vai com calma**
   - "Respira, vai dar certo"
   - O importante é aprender
   - Não ficar maluco

### Hackathons

**Benefícios**:

- ✅ Desenvolvimento rápido
- ✅ Aprendizado acelerado
- ✅ Networking
- ✅ Prêmios
- ✅ Viagens
- ✅ Portfolio

**Realidade**:
> "Para vocês irem bem em Hackathon, irem bem em competição e aprenderem de fato full stack development de Blockchain demora para caramba."

### Especialização vs Generalização

#### Especialista

- Muito bom em uma coisa
- Valioso para times
- Profundo conhecimento

#### Generalista (Jack of all trades, master of none)

- Sabe de tudo um pouco
- Menos profundidade
- Flexibilidade

> **Dica**: É bom aprender de tudo um pouco, mas ter uma especialidade forte.

---

## 14. Roadmap Visual Resumido

```
BLOCKCHAIN DEVELOPMENT ROADMAP
================================

📚 TREINAMENTO ZERO
   └─ Fundamentos, Bitcoin, Ethereum, DeFi, NFTs, MetaMask

      ┌──────────────┴──────────────┐
      ↓                             ↓

🎨 FRONT-END PATH              📝 SMART CONTRACTS PATH (Recomendado)
   │                              │
   ├─ HTML/CSS/JS                 ├─ Solidity Básico
   ├─ React                       ├─ Remix IDE
   ├─ 🟢 Portfólio 1              ├─ 🟢 FundMe
   ├─ Git                         ├─ Git
   ├─ React/TypeScript/Tailwind   ├─ Hardhat/Foundry
   ├─ Ethers.js                   ├─ Chainlink VRF/Keepers
   ├─ 🟢 Portfólio 2              ├─ Ethers.js
   ├─ 🟠 Uniswap Clone            ├─ 🟠 FundMe 2
   └─ Front-End Avançado          ├─ 🟠 Raffle.sol
                                  ├─ ERC-20, ERC-721, IPFS
                                  ├─ Aave, DAOs
                                  ├─ 🟢 Token Próprio
                                  ├─ 🟠 NFT Minter
                                  ├─ 🟠 Lend & Borrow
                                  └─ 🔴 NFT Marketplace

      ┌──────────────┬──────────────┘
      ↓              ↓

🔗 INTEGRAÇÃO (ESSENCIAL)
   └─ Ethers.js + APIs + Web3

🏆 HACKATHONS
   └─ Front + Smart Contracts + Integração

📈 AVANÇADO (Opcional)
   ├─ Security & Auditing
   ├─ Flash Loans & Arbitrage
   ├─ Bridges
   ├─ Outras Blockchains (Solana, Starknet, etc)
   └─ Back-end tradicional
```

---

## 15. Checklist de Progresso

### Fundamentos ✓

- [ ] Completar Treinamento Zero
- [ ] Instalar e usar MetaMask
- [ ] Fazer primeira transação na testnet
- [ ] Entender PoW vs PoS

### Git ✓

- [ ] Instalar Git
- [ ] Aprender comandos básicos
- [ ] Criar conta no GitHub
- [ ] Fazer primeiro commit

### Front-End ✓

- [ ] HTML básico
- [ ] CSS básico
- [ ] JavaScript (estudar direito!)
- [ ] React fundamentals
- [ ] 🟢 Portfólio 1
- [ ] Tailwind CSS
- [ ] TypeScript
- [ ] 🟢 Portfólio 2
- [ ] 🟠 Uniswap Clone (opcional)

### Smart Contracts ✓

- [ ] Solidity básico no Remix
- [ ] 🟢 FundMe
- [ ] Aprender Hardhat ou Foundry
- [ ] Chainlink Price Feeds
- [ ] Chainlink VRF
- [ ] 🟠 FundMe 2
- [ ] 🟠 Raffle.sol
- [ ] ERC-20 Token
- [ ] ERC-721 NFT
- [ ] IPFS
- [ ] 🔴 NFT Marketplace

### Integração ✓

- [ ] Ethers.js básico
- [ ] Conectar MetaMask ao site
- [ ] Chamar função de contrato do front-end
- [ ] Ler dados da blockchain
- [ ] Enviar transações

### Avançado ✓

- [ ] Security best practices
- [ ] Flash Loans
- [ ] Auditar contratos
- [ ] Explorar outras blockchains

---

## 16. Mensagem Final

> "Vale a pena investir sim. A grana que você pode ganhar nisso aqui é absurda. Mas não é nem por isso. É legal muito isso aqui. É muito da hora. A gente realmente gosta disso no ITA Blockchain Club."

> "É um ramo que tem muita chance para empreendedorismo. Para viajar para fora. Para ir para hackathon internacional com tudo pago. As chances são absurdas."

> "Só tem a ganhar aqui. Se dedicar, estudar muito isso aqui, você vai ter muita chance de brilhar."

### Contatos e Comunidade

- ITA Blockchain Club
- Participe de Hackathons
- Entre na comunidade
- Compartilhe conhecimento

- Vídeo original: <https://www.youtube.com/watch?v=RAThSWyMa8U>

---

**Última Atualização**: Baseado no material do ITA Blockchain Club
**Tempo Estimado**: 6+ meses de estudo dedicado
**Horas Semanais**: 10+ horas

**Boa sorte na sua jornada blockchain! 🚀**
