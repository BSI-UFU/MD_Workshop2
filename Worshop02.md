# WORKSHOP: Arquitetura Orientada a Eventos em Cenários de Alta Escala: Uma Análise de Padrões de Comunicação e Desacoplamento


## ⏱️ Workshop: EDA em Alta Escala (50 min)

### 1. Ponto de Partida: Context Mapping (10 min)

Utilizando a imagem fornecida como referência, traduzimos os domínios para um fluxo de eventos. O segredo da alta escala é o **Desacoplamento Temporal**: o emissor não espera o receptor.

**Mapeamento de Fluxo:**

* **Core Domain:** Emite eventos vitais (ex: `OrderPlaced`).
* **Supporting (A):** Reage para processar pagamentos ou validações.
* **Generic Subdomain:** Cuida de tarefas transversais como Notificações ou Logs.
* **External Context:** Sistemas legados ou APIs de terceiros que consomem dados via Webhooks/Adapters.

---

### 2. Análise de Padrões de Comunicação (15 min)

Para escalar, analisamos dois modelos principais de interação entre esses contextos:

| Padrão | Funcionamento | Nível de Desacoplamento |
| --- | --- | --- |
| **Coreografia** | Cada serviço observa o Broker e decide sua ação. | **Máximo.** Não há ponto central de falha lógica. |
| **Orquestração** | Um serviço central (Saga) comanda: "Execute A, depois B". | **Médio.** Facilita a visão do processo, mas cria dependência do orquestrador. |

**O Desafio da Alta Escala: Idempotência**
Em sistemas distribuídos, a rede falha. O padrão **At-least-once delivery** garante que a mensagem chegue, mas ela pode chegar repetida.

* **Solução:** O consumidor deve verificar se o ID da mensagem já foi processado antes de alterar o estado do banco.

---

### 3. Prática: Estrutura de Pastas e Implementação (20 min)

Para evitar que o código se torne um "Big Ball of Mud", a estrutura de pastas deve refletir o isolamento do Bounded Context e a infraestrutura de eventos.

**Estrutura de Pastas Sugerida:**

```text
/src
  /modules
    /sales-context (Core Domain)
      /domain
        /events        <-- Definição pura do evento (JSON Schema)
        /entities
      /application
        /handlers      <-- Lógica que dispara o evento
      /infrastructure
        /messaging     <-- Implementação (Kafka/RabbitMQ/AWS SNS)
        /outbox        <-- Persistência para garantir entrega (Outbox Pattern)
    /payment-context (Supporting A)
      /application
        /subscribers   <-- Escuta eventos do Sales Context

```

---

### 4. Conclusão e Trade-offs (5 min)

* **Vantagem:** Escalabilidade elástica. Se o `Supporting (A)` estiver lento, as mensagens acumulam no Broker sem derrubar o `Core Domain`.
* **Custo:** Consistência Eventual. O dado no `Generic Subdomain` pode estar alguns milissegundos (ou segundos) atrás do Core.

---

# MAPA DE CONTEXTO E FLUXO DE EVENTOS

```mermaid
graph TD
    %% Baseado no Core Domain da imagem fornecida
    Core[Bounded Context: Core Domain]

    %% Baseado no Supporting Subdomain (A) da imagem fornecida
    SuppA[Bounded Context: Supporting A]

    %% Baseado no Supporting Subdomain (B) da imagem fornecida
    SuppB[Bounded Context: Supporting B]

    %% Baseado no Generic Subdomain da imagem fornecida
    Generic[Bounded Context: Generic]

    %% Baseado no Bounded Context (External) da imagem fornecida
    External[Bounded Context: External]

    %% O Broker de Mensagens é a chave para a escala e desacoplamento
    Broker{Event Broker Message Bus}

    %% Fluxo de Publicação (Contexto emite para o Broker)
    Core -- Publica Evento 'PedidoCriado' --> Broker
    SuppA -- Publica Evento 'PagamentoConfirmado' --> Broker
    Generic -- Publica Evento 'NotificacaoEnviada' --> Broker

    %% Fluxo de Assinatura/Consumo (Contexto reage ao Broker)
    Broker -- Assina/Consome --> SuppB
    Broker -- Assina/Consome --> External
    Broker -- Assina/Consome --> SuppA
```
Este diagrama é a representação visual de como transformamos um design estratégico de **Domain-Driven Design (DDD)** em uma arquitetura reativa e resiliente. O foco aqui não é apenas "enviar mensagens", mas sim garantir que a **lógica de negócio** dite como a tecnologia escala.

## 1. Os Atores: Bounded Contexts

Cada caixa quadrada representa um **Contexto Delimitado**. No DDD, isso significa que cada um possui seu próprio banco de dados, seus próprios modelos de objetos e sua própria equipe de desenvolvimento.

* **Core Domain:** É o coração do negócio (ex: o motor de vendas ou reservas). Em alta escala, ele deve ser protegido de lentidões em outros sistemas. Por isso, ele "fala e esquece": publica o evento e continua operando.
* **Supporting (A e B):** São domínios auxiliares (ex: Processamento de Pagamento ou Controle de Estoque). Eles dão suporte ao Core, mas não são o negócio principal.
* **Generic Subdomain:** Funcionalidades que não são exclusivas da empresa (ex: Envio de E-mails ou Logs). Podem ser substituídas por serviços de terceiros sem afetar a lógica central.
* **External:** Representa sistemas fora do seu controle (APIs de parceiros, sistemas legados ou ERPs externos).


## 2. O Mediador: Event Broker (O Losango Central)

O **Event Broker** (como Kafka, RabbitMQ ou Pulsar) atua como o sistema nervoso central.

* **Desacoplamento Espacial:** O `Core Domain` não sabe que o `External` existe. Ele apenas sabe que deve entregar a mensagem ao Broker.
* **Desacoplamento Temporal:** Se o `SuppB` estiver fora do ar por 5 minutos devido a um pico de carga, o Broker armazena as mensagens. Quando o serviço voltar, ele processa tudo o que perdeu sem que o usuário final perceba um erro no `Core`.


## 3. Dinâmica de Fluxo (Setas)

O diagrama utiliza o padrão **Pub/Sub (Publish/Subscribe)**, que é dividido em duas fases críticas:

### A. Fluxo de Publicação (Upstream)

As setas que apontam **para** o Broker representam a saída de fatos que já ocorreram:

1. O `Core` emite `PedidoCriado`.
2. O `SuppA` emite `PagamentoConfirmado`.
3. O `Generic` emite `NotificacaoEnviada`.

> **Ponto de Alta Escala:** Note que o `Core` não chama o `SuppA` via HTTP. Se o fizesse, e o `SuppA` estivesse lento, a transação do `Core` ficaria aberta, consumindo memória e CPU desnecessariamente.

### B. Fluxo de Assinatura/Consumo (Downstream)

As setas que saem **do** Broker representam a reação aos fatos:

1. O `SuppB` (ex: Logística) está "ouvindo" o Broker. Assim que o `PedidoCriado` aparece, ele começa a separar o pacote.
2. O `External` pode estar consumindo esses dados para alimentar um Dashboard de Business Intelligence em tempo real.
3. O `SuppA` (Pagamentos) pode consumir eventos de outros contextos para liberar estornos ou créditos.


## 4. Análise de Desacoplamento Técnica

Para que este diagrama funcione na prática de alta escala, aplicamos três conceitos:

1. **Imutabilidade:** Uma vez que o evento `PedidoCriado` saiu do `Core`, ele é um fato histórico. Não pode ser alterado, apenas compensado por outro evento (ex: `PedidoCancelado`).
2. **Fan-out:** O Broker permite que **um** evento (PedidoCriado) seja entregue simultaneamente para **N** interessados (Logística, BI, Notificação) sem que o emissor precise fazer N chamadas.
3. **Backpressure:** Se o sistema `External` for lento, ele consome as mensagens no seu próprio ritmo, sem "engasgar" a produção de mensagens dos outros contextos.

---

# BROKE E ACL


## 1. O Papel da Camada Broker em Alta Escala

Em cenários de alta escala, o Broker não é apenas um "correio". Ele atua como um **buffer de resiliência** e um **distribuidor de carga**.

### Atributos Técnicos da Camada:

* **Persistência (Log Replay):** Diferente de uma chamada HTTP, o Broker (como Kafka) armazena o evento. Se o consumidor cair, ele pode ler do ponto onde parou.
* **Backpressure:** Se o *Core Domain* gera 10.000 eventos/segundo e o *Supporting (B)* só aguenta 2.000, o Broker retém o excesso, evitando o "cascateamento de falhas" (o efeito dominó).
* **Fan-out:** A capacidade de entregar a mesma mensagem para múltiplos interessados com latência mínima.

---

## 2. O Broker pode ser considerado uma ACL?

**Tecnicamente, não.** O Broker é infraestrutura (o "cano"). A **ACL** é um padrão de design de software (a "lógica").

No entanto, no contexto de **Event-Driven Architecture (EDA)**, a confusão é comum porque a ACL é implementada **nas extremidades** do Broker.

### Como a ACL funciona com o Broker:

Uma **Anti-Corruption Layer (Camada Anticorrupção)** serve para impedir que a semântica de um sistema externo "corrompa" o seu modelo de domínio interno.

* **ACL no Produtor (Outbound):** Antes de enviar uma mensagem para o Broker, o seu serviço traduz o seu objeto interno para um **Contrato Público (Schema)**. Isso evita que mudanças internas quebrem os outros.
* **ACL no Consumidor (Inbound):** Quando o *Supporting (B)* recebe um evento do *Core Domain*, ele passa por uma camada que traduz esse evento para a linguagem que o *Supporting (B)* entende.

> **Exemplo Prático:** Se o sistema *External* (um ERP legado) envia dados com nomes de campos confusos como `CD_CLI_01`, sua ACL no consumidor traduz isso para `CustomerIdentifier` antes de chegar na sua lógica de negócio.

---

## 3. Onde a "Magia" acontece: Schema Registry

Para que o Broker ajude no desacoplamento sem virar um caos, usamos um **Schema Registry**.

1. O **Produtor** consulta o Registry para garantir que o evento segue a regra.
2. O **Broker** transporta o evento (geralmente em formato binário como Avro ou Protobuf para performance).
3. O **Consumidor** valida o Schema.

**Isso é a ACL em nível de contrato:** Você garante que ninguém "suje" o barramento com dados malformados.

---

## 4. Resumo da Estrutura (Padrão de Comunicação)

Para o seu workshop, você pode definir a relação assim:

| Componente | Função no Workshop |
| --- | --- |
| **Broker** | Infraestrutura de transporte assíncrono e garantia de entrega. |
| **ACL** | Lógica de tradução que protege o Bounded Context de modelos externos. |
| **Event Schema** | O "contrato" diplomático entre os contextos que a ACL deve respeitar. |


## Diagrama: O Fluxo da ACL no Mundo de Eventos

```mermaid
graph LR
    subgraph "Ecossistema Externo"
        Ext[Sistema Legado / ERP]
    end

    subgraph "Camada de Fronteira (ACL)"
        Adapter[Adaptador de Entrada]
        Translator[Tradutor de Esquema]
    end

    subgraph "Infraestrutura de Mensageria"
        Broker{Event Broker / Kafka}
    end

    subgraph "Seu Bounded Context (Core)"
        Domain[Core Domain Logic]
        Consumer[Consumidor Interno]
    end

    %% Fluxo de Dados
    Ext -- "Envia XML/JSON 'sujo' (ex: CD_CLI_01)" --> Adapter
    Adapter -- "Normaliza Dados" --> Translator
    Translator -- "Publica Evento Limpo (ex: CustomerID)" --> Broker
    Broker -- "Entrega Evento Padronizado" --> Consumer
    Consumer -- "Executa Regra de Negócio" --> Domain

```

## Detalhamento dos Componentes da ACL

**3 pontos vitais do diagrama:**

### 1. O Adaptador (Interface de Entrada)

É o componente que "fala a língua" do vizinho. Se o sistema externo só envia arquivos CSV por FTP ou chamadas Webhook confusas, o **Adaptador** é quem recebe esse impacto. Ele isola a tecnologia de transporte externa da sua arquitetura.

### 2. O Tradutor (A Essência da ACL)

Aqui ocorre a **Tradução Semântica**.

* **De:** Um modelo de dados confuso, com nomes de campos técnicos ou em outra língua.
* **Para:** O seu **Linguagem Ubíqua** (ex: `OrderPlaced`, `ClientVerified`).
* **Importante:** Se o sistema externo mudar uma coluna no banco de dados deles, você só altera o **Tradutor**. Seu Core Domain nem percebe a mudança.

### 3. O Contrato no Broker

Uma vez que o dado passa pela ACL, ele entra no **Broker** seguindo um contrato rigoroso (Schema). Isso garante que qualquer outro serviço da sua empresa que consuma esse evento receba um dado confiável e de alta qualidade.


## Conclusão para o Workshop: O Broker é a ACL?

Como vimos, o **Broker é o canal de transporte seguro**, mas a **ACL é o "filtro"** que garante que apenas informações úteis e bem formatadas entrem nesse canal. Sem a ACL, seu Broker vira um "pântano de dados" (Data Swamp).

---

# DESAFIO BLACK FRIDAY


```mermaid
graph LR
    %% Definição dos Contextos
    subgraph Core_Domain [Core Domain: Gestão de Pedidos]
        OrderApp[Serviço de Pedidos]
    end

    subgraph Messaging [Camada de Mensageria: O Mediador]
        Broker{Event Broker / Kafka}
        Registry[Schema Registry]
    end

    subgraph Supporting [Supporting Context]
        Stock[Serviço de Estoque]
    end

    subgraph Legacy_Protection [Camada de Fronteira]
        ACL[Anti-Corruption Layer / Worker]
    end

    subgraph External [External Context: Legado Fiscal]
        FiscalDB[(DB Legado)]
    end

    %% Fluxo de Dados e Dinâmica
    OrderApp -- "1. Publica 'OrderPlaced' (Alta Escala)" --> Broker
    Broker -. "Valida Contrato" .-> Registry

    %% Consumo Desacoplado
    Broker -- "2a. Consumo em Tempo Real" --> Stock
    
    %% Fluxo com ACL e Throttling
    Broker -- "2b. Consumo Assíncrono" --> ACL
    ACL -- "3. Tradução Semântica & Throttling (Carga Controlada)" --> FiscalDB

    %% Estilização para clareza
    style Broker fill:#f96,stroke:#333,stroke-width:2px
    style ACL fill:#bbf,stroke:#333,stroke-width:2px
    style OrderApp fill:#dfd
    style FiscalDB fill:#ffd

```

### 🧠 Guia de Leitura para os Participantes:

1. **O Mediador (Broker):** Funciona como o amortecedor. Ele aceita 10k pedidos por segundo do **Core Domain**, mesmo que os outros sistemas sejam lentos.
2. **A Dinâmica de Fluxo:** * O **Estoque** consome as mensagens rapidamente (escala paralela ao Core).
* A **ACL** consome as mesmas mensagens, mas atua como um "redutor de velocidade" (Throttling) para não derrubar o **Legado Fiscal**.

3. **A ACL em Ação:** Observe que ela é a única que "toca" o sistema legado. Ela traduz os dados modernos do seu sistema para o formato esperado pelo banco de dados antigo (ex: convertendo JSON para a estrutura de campos em latim).

Este exercício foi desenhado para ser a "prova de fogo" do workshop. Ele força os participantes a saírem da teoria e tomarem decisões arquiteturais reais de **alta escala**, focando em como o **Broker** e a **ACL** protegem o domínio.

---

## 🕒 Exercício Prático: O Desafio da Black Friday (20 min)

### O Cenário

A empresa **"Global-Log"** possui um sistema de **Gestão de Pedidos (Core Domain)**. Durante a Black Friday, o volume de vendas aumenta 50x. Atualmente, o sistema de pedidos tenta avisar o **Sistema de Notas Fiscais (Externo/Legado)** via API síncrona, mas o Legado trava com o volume, derrubando as vendas.

### O Material de Partida

Os participantes recebem o seguinte mapeamento de contexto simplificado:

1. **Core Domain (Pedidos):** Gera 10k eventos/seg. Não pode esperar ninguém.
2. **External Context (Legado Fiscal):** Só aguenta 100 requisições/seg. Usa nomes de campos em latim (ex: `id_venditio`).
3. **Supporting Context (Estoque):** Precisa saber das vendas para baixar o inventário.


### A Tarefa (Trabalho em Grupo ou Individual)

Os participantes devem desenhar (em papel ou ferramenta digital) e justificar:

#### 1. Propor o Mediador (5 min)

* Qual tecnologia/padrão de **Broker** escolheriam para suportar esse "buffer" de mensagens?
* Como garantir que o **Core Domain** não fique travado esperando o **Legado**?

#### 2. Desenhar a Dinâmica do Fluxo e ACL (10 min)

* Onde exatamente você colocaria a **ACL** (Anti-Corruption Layer)?
* Desenhe o fluxo da mensagem desde a criação do pedido até a chegada no Legado, indicando a transformação dos dados (Tradução).
* Como o **Estoque** se encaixa nesse fluxo sem acoplar com o **Legado**?

#### 3. Definir a Estratégia de Escala (5 min)

* Se o **Legado** cair por 2 horas, o que acontece com as mensagens no Broker?
* Como evitar que o **Legado** processe a mesma nota fiscal duas vezes (Idempotência)?


## 📝 Guia de Resolução (Para o Facilitador)

Ao final dos 20 minutos, apresente esta solução esperada para comparação:

* **O Mediador:** Um Broker com suporte a **Backpressure** (ex: Kafka ou RabbitMQ com filas persistentes). O Core publica e esquece.
* **A ACL:** Deve ficar na frente do **Legado**. Ela consome do Broker, traduz os nomes dos campos (de `OrderID` para `id_venditio`) e faz o "Throttling" (entrega as mensagens devagar, no ritmo que o legado aguenta).
* **O Fluxo:** * `Pedido` -> `Broker` (Evento: `OrderPlaced`)
* `Broker` -> `Estoque` (Consome em tempo real)
* `Broker` -> `ACL` -> `Legado Fiscal` (Consome de forma controlada/lenta)

### Checklist de Avaliação para os Participantes

> * [ ] O Core Domain ficou livre de chamadas HTTP externas?
> * [ ] O Legado está protegido por uma camada de tradução (ACL)?
> * [ ] O sistema aguenta a queda de um dos componentes sem perder dados?
> 
> 

---

# 📄 Resumo Executivo: Solução para Alta Escala (Template)

**Nome do Grupo/Participante:** _______________________________________
**Desafio:** Integração Core Domain (Alta Performance) vs. Legado Fiscal (Baixa Resiliência).

### 1. Identificação da Dinâmica de Fluxo

* **Padrão de Comunicação Adotado:** ( ) Coreografia | ( ) Orquestração
* **Justificativa:** __________________________________________________________________
*(Dica: Por que o Core Domain não deve saber que o Fiscal existe?)*

### 2. O Papel do Mediador (Broker)

* **Tecnologia Proposta:** __________________________________________________________
* **Estratégia de Retenção:** Se o Legado ficar offline por 4 horas, como o Broker deve se comportar?
* ( ) Descartar mensagens antigas.
* ( ) Armazenar em disco/log para processamento posterior (Backlog).



### 3. A Camada de Fronteira (ACL)

* **Responsabilidades da sua ACL (Descreva 2):**
1. ---


2. ---




* **Tradução Semântica:** Como você mapearia o campo `customer_id` do Core para o Legado?
* `ID_INTERNO` -> `____________________`



### 4. Garantia de Desacoplamento e Idempotência

* **Problema:** O Broker entregou a mesma mensagem duas vezes para a ACL.
* **Solução Proposta:** ______________________________________________________________
*(Dica: Como a ACL pode verificar se aquela Nota Fiscal já foi enviada antes de tentar de novo?)*

---

## 🏁 Encerramento do Workshop (Últimos 5 min)

Para finalizar, apresente este fechamento focado em **Trade-offs**:

> "Arquitetura Orientada a Eventos não é 'bala de prata'. Ganhamos **escalabilidade infinita** e **isolamento de falhas**, mas pagamos com **complexidade de rastreabilidade**. O sucesso não está em usar a ferramenta mais cara, mas em garantir que o seu **Domínio** nunca seja corrompido pela lentidão ou pela semântica de sistemas externos."

---

**Gostaria que eu formatasse este workshop completo em um documento PDF ou Markdown estruturado para você distribuir aos participantes?**
