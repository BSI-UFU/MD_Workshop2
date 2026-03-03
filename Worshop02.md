# WORKSHOP: Arquitetura Orientada a Eventos em Cenários de Alta Escala: Uma Análise de Padrões de Comunicação e Desacoplamento


Este workshop foi desenhado para ser intensivo, prático e focado na transição de um design estratégico (DDD) para uma implementação técnica escalável utilizando **Event-Driven Architecture (EDA)**.

---

## ⏱️ Cronograma do Workshop (50 min)

| Tempo | Atividade | Foco |
| --- | --- | --- |
| **00-10 min** | Alinhamento Estratégico | Do Context Map aos Eventos de Domínio. |
| **10-25 min** | Padrões de Comunicação | Choreography vs. Orchestration e Event Delivery. |
| **25-45 min** | Mão na Massa (Prática) | Estrutura de pastas, idempotência e desacoplamento. |
| **45-50 min** | Q&A e Conclusão | Trade-offs e próximos passos. |

---

## 1. O Ponto de Partida: Context Mapping (10 min)

Para este cenário, assumimos um sistema de **E-commerce Global**. O mapeamento de contextos (Bounded Contexts) já foi definido para garantir que cada serviço possua sua própria linguagem ubíqua e limites de dados claros.

### Por que Eventos aqui?

Em alta escala, a comunicação síncrona (HTTP/REST) entre esses contextos cria um "acoplamento temporal". Se o serviço de *Pagamento* cai, o *Pedido* falha. Com eventos, o **Pedido** apenas publica `OrderPlaced` e segue sua vida.

---

## 2. Padrões de Comunicação e Desacoplamento (15 min)

### Coreografia vs. Orquestração

* **Coreografia:** Cada serviço sabe o que fazer quando ouve um evento. É o ápice do desacoplamento, ideal para alta escala.
* **Orquestração:** Um "maestro" central coordena as chamadas. Útil em fluxos de compensação complexos (Sagas), mas pode virar um gargalo.

### Garantias de Entrega e Idempotência

Em cenários de alta escala, "exatamente uma entrega" é um mito caro. Trabalhamos com **At-least-once delivery**.

* **Desafio:** O consumidor pode receber o mesmo evento duas vezes.
* **Solução:** Implementar o padrão **Idempotent Consumer** e **Outbox Pattern** para garantir que o banco de dados e a mensagem estejam em sincronia.

---

## 3. Estrutura de Projeto e Implementação (20 min)

Para manter o desacoplamento dentro do código, utilizamos uma arquitetura limpa (Clean/Hexagonal) onde os eventos são cidadãos de primeira classe.

### Estrutura de Pastas Sugerida

```text
src/
└── Modules/
    └── Ordering/
        ├── Domain/
        │   ├── Events/          # Definição dos Eventos (ex: OrderCreated.ts)
        │   ├── Entities/
        │   └── Services/
        ├── Application/
        │   ├── Commands/        # Orquestra a lógica de negócio
        │   └── EventHandlers/   # Reage a eventos de outros contextos
        └── Infrastructure/
            ├── Messaging/       # Implementação (Kafka, RabbitMQ, SNS/SQS)
            │   ├── Publishers/
            │   └── Consumers/
            └── Persistence/     # Repositórios e Outbox

```

### Prática: O Fluxo do Evento

1. **Domain Event:** Ocorre uma mudança de estado na Entidade.
2. **Outbox:** O evento é salvo na mesma transação do banco de dados.
3. **Relay/Publisher:** Um processo separado lê o Outbox e publica no Broker (ex: Kafka).
4. **Subscriber:** O serviço de *Estoque* ou *Logística* consome o evento de forma assíncrona.

---

## 4. Análise de Trade-offs (5 min)

* **Prós:** Escalabilidade horizontal massiva, resiliência a falhas parciais, evolução independente de serviços.
* **Contras:** Consistência eventual (o dado demora milisegundos para propagar), depuração mais complexa (rastreabilidade via *Correlation IDs* é obrigatória).

---

### Conclusão e Próximo Passo

O desacoplamento real não vem apenas de "usar um broker", mas de garantir que os contextos não compartilhem esquemas de banco de dados e que a lógica de negócio seja resiliente a duplicidade de mensagens.

**Gostaria que eu detalhasse a implementação de um "Idempotent Consumer" em uma linguagem específica (Node.js, Java ou Go) para este cenário?**

---

# MAPA DE CONTEXTO E FLUXO DE EVENTOS

```mermaid
graph TD
    %% Baseado no Core Domain da imagem
    Core

    %% Baseado no Supporting Subdomain (A) da imagem
    SuppA

    %% Baseado no Supporting Subdomain (B) da imagem
    SuppB

    %% Baseado no Generic Subdomain da imagem
    Generic

    %% Baseado no Bounded Context (External) da imagem
    External

    %% O Broker de Mensagens é a chave para a escala
    Broker{cite: Event Broker (Message Bus)}

    %% Fluxo de Publicação e Assinatura
    Core -- cite: Publica 'PedidoCriado' --> Broker
    SuppA -- cite: Publica 'PagamentoConfirmado' --> Broker
    Generic -- cite: Publica 'LogAlterado' --> Broker

    Broker -- cite: Assina/Consome --> SuppB
    Broker -- cite: Assina/Consome --> External
    Broker -- cite: Assina/Consome --> SuppA
```