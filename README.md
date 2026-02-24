# **Arquitetura Orientada a Eventos em Cenários de Alta Escala: Uma Análise de Padrões de Comunicação e Desacoplamento**
---

## Estrutura do Workshop (50 Minutos)

### 1. Introdução: O Problema do Acoplamento (05 min)

* **O Monólito vs. Microsserviços Síncronos:** Por que o HTTP encadeado (chamadas REST em cascata) é o "inimigo" da alta escala.
* **Definição de Evento:** A diferença entre um comando (*"Crie este pedido"*) e um evento (*"Pedido Criado"*).
* **Mudança de Mentalidade:** Do fluxo de controle para o fluxo de dados.

### 2. Padrões de Comunicação e Topologias (15 min)

* **Pub/Sub (Publish/Subscribe):** O papel do *Broker* (Kafka, RabbitMQ, AWS EventBridge).
* **Event Streaming vs. Message Queuing:** Quando usar cada um.
* **Topologias de Orquestração:**
* **Broker Topology:** Coreografia (os serviços reagem organicamente).
* **Mediator Topology:** Orquestração (um controlador central coordena os passos).



### 3. Estratégias para Desacoplamento e Consistência (15 min)

* **Eventual Consistency:** Como lidar com o fato de que os dados não estarão iguais em todos os lugares ao mesmo tempo.
* **Padrão Outbox:** Garantindo que o banco de dados e a mensagem para o broker sejam atualizados de forma atômica.
* **CQRS (Command Query Responsibility Segregation):** Separando a escrita da leitura para ganhar performance extrema.
* **Idempotência:** Por que todo consumidor de eventos deve estar preparado para receber a mesma mensagem duas vezes.

### 4. Cenários de Alta Escala e Desafios Reais (10 min)

* **Backpressure:** O que fazer quando o produtor é mais rápido que o consumidor.
* **DLQ (Dead Letter Queues):** Tratamento de erros sem interromper o fluxo principal.
* **Observabilidade:** O desafio de rastrear uma transação que atravessa múltiplos serviços (Tracing Distribuído).

### 5. Conclusão e Q&A (05 min)

* Recapitulando os benefícios: Escalabilidade agnóstica, tolerância a falhas e agilidade no desenvolvimento.
* Espaço para perguntas rápidas.

---

## Dicas para o Facilitador

* **Use uma analogia do mundo real:** O exemplo da "Cozinha de Restaurante" funciona muito bem. O garçom deixa o pedido (evento) no balcão e volta a atender. O cozinheiro pega o pedido quando pode. Não há ninguém "esperando parado".
* **Foco no "Porquê":** Em 50 minutos, você não conseguirá ensinar a configurar um cluster Kafka. Foque em **quando** escolher cada padrão e quais os *trade-offs* (custo de complexidade vs. ganho de escala).
* **Checklist de Prontidão:** Termine com 3 perguntas que o arquiteto deve se fazer antes de adotar EDA (ex: "Meu negócio tolera consistência eventual?").


# **"Kit de Sobrevivência do Arquiteto EDA"**.

Aqui estão os detalhes dos tópicos críticos e as recomendações de ferramentas para apresentação:

## 1. Detalhando o Padrão "Transactional Outbox"

Este é o tópico mais importante para evitar a perda de dados. Em sistemas de alta escala, você não pode simplesmente salvar no banco e enviar uma mensagem para o Kafka em seguida (se o Kafka falhar, seu banco fica inconsistente).

* **Como funciona:** O serviço grava o estado da entidade **e** o evento em uma tabela de "Outbox" dentro da mesma transação de banco de dados.
* **O Relay:** Um processo separado (como o **Debezium**) lê os logs do banco e publica no Broker. Isso garante que, se gravou no banco, a mensagem *será* enviada.

---

## 2. Bibliografia Recomendada (O "Caminho das Pedras")

Sugira estes três livros como os pilares para quem quer se aprofundar:

1. **"Designing Data-Intensive Applications" (Martin Kleppmann):** A "Bíblia" moderna. Fundamental para entender sistemas distribuídos e *streams*.
2. **"Building Microservices" (Sam Newman):** Excelente para entender o desacoplamento e as fronteiras de contexto (Bounded Contexts).
3. **"Enterprise Integration Patterns" (Gregor Hohpe):** O catálogo clássico de como mensagens devem fluir entre sistemas.

---

## 3. Stack Tecnológica Sugerida (Comparativo)

Apresente uma tabela rápida para ajudar os participantes a escolherem suas ferramentas:

| Categoria | Ferramenta | Por que usar? |
| --- | --- | --- |
| **Log-based Broker** | **Apache Kafka** | Ideal para altíssimo rendimento e retenção de histórico (event sourcing). |
| **Cloud Native** | **AWS EventBridge** | Perfeito para arquiteturas *serverless* e integração rápida entre serviços AWS. |
| **Tradicional/Task** | **RabbitMQ** | Excelente para roteamento complexo e filas de prioridade. |
| **Observabilidade** | **Jaeger / Honeycomb** | Essencial para fazer o *Distributed Tracing* e não se perder nos eventos. |

---

## 4. O Checklist de Decisão (The "Go/No-Go" List)

Entregue isso como um slide de fechamento. O arquiteto deve adotar EDA se:

* [ ] O sistema precisa de **escalabilidade horizontal** independente para cada parte.
* [ ] O negócio aceita **consistência eventual** (ex: o saldo do usuário demora 1s para atualizar após a compra).
* [ ] Existe a necessidade de **reprocessamento** de dados históricos.
* [ ] A latência de uma resposta síncrona (esperar todos os serviços responderem) está prejudicando a UX.

---

# **E-commerce** 
---

## Estudo de Caso: "Do Clique ao Pacote"

**Cenário:** Uma Black Friday onde o sistema recebe 10.000 pedidos por segundo. No modelo REST tradicional (síncrono), se o serviço de *E-mail* cair, a compra falha. No modelo **EDA**, o sistema sobrevive.

### 1. O Gatilho: Evento "Pedido Realizado" (`OrderPlaced`)

O usuário clica em "Comprar". O **Serviço de Pedidos** valida o cartão e grava no banco. Em vez de chamar 5 APIs diferentes, ele apenas publica:

> *"Atenção: O Pedido #123 foi realizado pelo Cliente X."*

### 2. A Reação em Cadeia (Coreografia)

Vários serviços estão "escutando" o Broker (Kafka/RabbitMQ) e reagem de forma independente:

* **Serviço de Estoque:** Decrementa os itens. Se acabar, gera um evento `OutOfStock`.
* **Serviço de Pagamento:** Processa a transação com o gateway. Gera `PaymentConfirmed` ou `PaymentFailed`.
* **Serviço de Notificação:** Envia o e-mail de confirmação (sem travar a jornada do usuário).
* **Serviço de Logística:** Começa a preparar a etiqueta de envio assim que o pagamento é confirmado.

---

### 3. Visualizando o Desacoplamento (Tabela de Fluxo)

| Ação | Modelo Síncrono (Acoplado) | Modelo EDA (Desacoplado) |
| --- | --- | --- |
| **Falha no E-mail** | O checkout trava ou dá erro 500 para o cliente. | O pedido é processado. O e-mail sai 2 min depois, quando o serviço voltar. |
| **Novo Serviço** | Precisa alterar o código do "Serviço de Pedidos" para chamar a nova API. | O novo serviço (ex: *Anti-fraude*) apenas começa a "ouvir" o evento existente. |
| **Pico de Acesso** | Todos os serviços precisam escalar juntos para não derrubar o sistema. | O Broker amortece o tráfego. Os serviços consomem no seu próprio ritmo (**Backpressure**). |

---

### 4. O Desafio Técnico: A "Transação Distribuída"

Neste ponto do workshop, você deve provocar a audiência:

> *"E se o pagamento for aprovado, mas o estoque falhar?"*

* **Solução EDA:** O padrão **Saga (Orquestrada ou Coreografada)**.
* **Ação:** O Serviço de Estoque emite um evento de erro, e o Serviço de Pagamento, ao ouvir isso, realiza o estorno automaticamente (**Compensação**).

---

### 5. Insight de Ouro para o Workshop

Destaque que, em alta escala, o banco de dados de **Leitura** não precisa ser o mesmo da **Escrita** (CQRS).

* **Escrita:** Relacional (PostgreSQL/MySQL) para garantir a integridade do pedido.
* **Leitura:** NoSQL (Elasticsearch/Redis) alimentado por eventos para que a busca de produtos seja instantânea, mesmo com milhões de acessos.

---

### Resumo para o seu Slide:

1. **Produtor:** Microserviço de Checkout.
2. **Evento:** `OrderCreated`.
3. **Consumidores:** Estoque, Pagamento, CRM, BI, Logística.
4. **Resultado:** Sistema resiliente, onde partes podem falhar sem derrubar o todo.


# Perguntas e respostas

---

## 1. "Como lidar com a Consistência Eventual se o usuário exige ver o dado atualizado na hora?"

* **Resposta Curta:** Use **UI/UX Optimística** ou **Sticky Sessions**.
* **Aprofundamento:** No E-commerce, quando o usuário clica em "Comprar", a UI mostra "Processando" ou "Pedido Recebido". O backend não precisa confirmar o estoque *naquele milissegundo* para responder o HTTP. Se a consistência for crítica (ex: transferência bancária), EDA pode não ser a primeira escolha para o *core* da transação, mas sim para os efeitos colaterais (notificações, extratos).

## 2. "E se a ordem das mensagens importar? (Ex: Cancelar antes de Criar)"

* **Resposta Curta:** Use **Partition Keys** (Chaves de Partição).
* **Aprofundamento:** No Kafka, por exemplo, se você enviar todas as mensagens do `Pedido #123` com a mesma chave, elas cairão na mesma partição e serão processadas na ordem exata de envio. Fora isso, o padrão **Saga** ajuda a gerenciar estados complexos onde a ordem dita o sucesso da operação.

## 3. "Como depurar (debug) um erro que acontece entre 5 serviços diferentes?"

* **Resposta Curta:** **Correlation IDs** e **Distributed Tracing**.
* **Aprofundamento:** Você deve injetar um ID único no primeiro evento. Todas as logs e sub-eventos gerados a partir dali carregam esse ID. Ferramentas como **OpenTelemetry**, **Jaeger** ou **Zipkin** permitem visualizar o "mapa" de por onde a mensagem passou e onde ela travou. Sem isso, você está no escuro.

## 4. "O que acontece se o consumidor processar a mesma mensagem duas vezes?"

* **Resposta Curta:** Design de **Idempotência**.
* **Aprofundamento:** Falhas de rede acontecem. O consumidor deve ser "inteligente". Antes de processar, ele checa no banco: *"Eu já processei o Pedido #123?"*. Se sim, ele descarta a mensagem e confirma o recebimento (ACK). Nunca confie que o Broker entregará a mensagem "exatamente uma vez" em alta escala.

## 5. "EDA não torna o sistema complexo demais para times pequenos?"

* **Resposta Curta:** Sim, o "imposto da complexidade" é real.
* **Aprofundamento:** EDA resolve problemas de **escala** e **acoplamento**. Se o seu sistema tem pouco tráfego ou apenas 2 microserviços, um monólito bem feito ou chamadas REST síncronas são melhores. EDA é um investimento para quando o custo da rigidez do sistema supera o custo de gerenciar a infraestrutura de eventos.

---

### Dica de Ouro para o Workshop:

Sempre que alguém fizer uma pergunta sobre "E se falhar?", responda com: **"Em sistemas distribuídos, a falha não é uma possibilidade, é uma certeza. EDA é sobre como falhar com elegância sem derrubar o prédio inteiro."**

---

