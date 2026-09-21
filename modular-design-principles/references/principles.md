# Princípios em profundidade

Uma seção por princípio: **definição**, **regras para agentes**, **exemplo abstrato**. Sem suposições de stack ou pastas.

---

## 1 — Fronteiras bem definidas

**Definição.** Consumidores dependem de uma **superfície pública pequena e intencional** (operações, eventos, tipos que fazem parte do contrato). Todo o resto é detalhe de implementação.

**Regras para agentes.**

- Prefira estender comportamento adicionando à API **documentada** em vez de importar internals.
- Ao sugerir refatorações, preserve ou reduza a superfície pública; não a amplie "por conveniência".
- Nomeie as coisas para que **contrato vs interno** seja óbvio em reviews (ex.: "operação pública" vs "helper interno" é uma distinção conceitual mesmo sem ferramentas).

**Exemplo abstrato.** Um contexto de "Checkout" expõe `placeOrder(command)` e eventos `OrderPlaced`. Outros contextos não devem alcançar as tabelas internas de preços do Checkout; eles assinam eventos ou chamam `placeOrder`, não "atualizam a linha X".

---

## 2 — Composabilidade

**Definição.** Módulos podem ser **montados em produtos ou implantações diferentes** sem reescrever sua lógica central para cada combinação.

**Regras para agentes.**

- Evite suposições ocultas como "isso só roda quando o módulo B está presente", a menos que sejam expressas como **integração opcional** ou contrato de **plugin**.
- Configuração e feature flags não devem virar spaghetti que apenas uma implantação entende.

**Exemplo abstrato.** O mesmo módulo "Inventory" funciona em uma ferramenta CLI pequena e em um app web grande porque seu contrato não assume uma UI ou host específico — apenas a raiz de composição muda.

---

## 3 — Independência

**Definição.** Módulos não dependem de **estado mutável compartilhado oculto** entre fronteiras. Testes podem rodar um módulo com **fakes** nas suas bordas.

**Regras para agentes.**

- Sinalize "singletons globais" que codificam política entre módulos sem um contrato explícito.
- Prefira **passar dependências explicitamente** ou **injeção declarada** a globais ambientes para preocupações transversais.

**Exemplo abstrato.** Dois serviços em módulos diferentes mutam ambos um cache de escopo de processo chaveado por "user id" sem coordenação → a independência é violada; substitua por uma interface de cache explícita possuída por um módulo ou um serviço compartilhado documentado.

---

## 4 — Escala individual

**Definição.** **Throughput, armazenamento, batching e limites** podem ser ajustados por módulo quando necessário, sem forçar uma configuração global a todos.

**Regras para agentes.**

- Ao fazer tuning de performance, pergunte **qual contexto delimitado** possui o gargalo; evite "consertar" acoplando caminhos de código não relacionados.
- Sugira quotas, pools ou tamanhos de batch **por módulo** quando os perfis de carga diferirem.

**Exemplo abstrato.** "Search" precisa de uma réplica de leitura grande e cacheamento agressivo; "Billing" precisa de escritas seriais estritas. As políticas de escala não são idênticas, e nenhum módulo força suas configurações sobre o outro.

---

## 5 — Comunicação explícita

**Definição.** Toda interação **entre módulos** passa por **contratos conhecidos**: APIs, mensagens, eventos ou schemas versionados — não arquivos compartilhados incidentais ou canais laterais implícitos.

**Regras para agentes.**

- Documente **entradas, saídas, erros e versionamento** para tudo que cruza uma fronteira.
- Desencoraje "é só importar este DTO do pacote deles" quando esse DTO é na verdade uma forma de **persistência interna**.

**Exemplo abstrato.** O módulo A notifica o módulo B via `OrderPlaced { orderId, placedAt }` em um bus, não escrevendo no banco de dados de B "porque é mais rápido".

---

## 6 — Substituibilidade

**Definição.** Dependências de outros módulos são expressas em termos de **interfaces, protocolos ou contratos estáveis** para que implementações possam ser trocadas ou mockadas.

**Regras para agentes.**

- Nas fronteiras, prefira **interfaces estreitas** ("gateway de pagamento", "clock", "gerador de IDs") a tipos concretos de vendors vazando para dentro.
- Refatorações que **prendem** um módulo a uma única tecnologia em todos os lugares devem ser questionadas, a menos que seja uma escolha deliberada de plataforma.

**Exemplo abstrato.** "Notifications" depende de `Notifier` com `send(recipient, body)`; email vs SMS vs push é substituível atrás dessa porta.

---

## 7 — Independência de implantação

**Definição.** O código dos módulos não **assume** co-localização no mesmo processo ou release, a menos que isso seja uma decisão arquitetural **explícita**.

**Regras para agentes.**

- Evite "chamar essa função diretamente no pacote deles" como a única história de integração quando múltiplas implantações são possíveis.
- Prefira contratos que funcionem com entrega **in-process, out-of-process ou assíncrona** com mudança mínima.

**Exemplo abstrato.** A mesma lógica de domínio pode rodar em um monolito hoje e atrás de uma fila de mensagens amanhã, porque as interações foram modeladas como operações/eventos, não como singletons in-process fixados no código.

---

## 8 — Isolamento de estado

**Definição.** Cada módulo **possui** seu store autoritativo e a nomenclatura de seus fatos. Nenhum compartilhamento silencioso dos mesmos dados lógicos entre fronteiras sem uma **regra clara** (quem escreve, quem lê, como a consistência é alcançada).

**Regras para agentes.**

- Trate **persistência por alcançamento** (ler/escrever diretamente o store de outro módulo) como um **design smell**, a menos que documentado como padrão excepcional revisado.
- Exija **nomes inequívocos** para conceitos persistidos quando múltiplos módulos têm substantivos semelhantes.

**Exemplo abstrato.** "Customer" no CRM e "Customer" no Billing são agregados diferentes, com IDs diferentes ou mapeamento explícito — não dois módulos atualizando uma linha ambígua de `customers`.

---

## 9 — Observabilidade

**Definição.** Logs, métricas, traces e health checks podem ser **atribuídos** a um módulo (e frequentemente a um caso de uso) para que incidentes sejam diagnosticáveis sem ler o sistema inteiro.

**Regras para agentes.**

- Ao adicionar diagnósticos, inclua **contexto** (qual operação, qual correlation id), não apenas "aconteceu um erro".
- Evite linhas de log que **não podem** ser filtradas por equipe responsável ou subsistema.

**Exemplo abstrato.** Um pagamento falho mostra o span `billing.capture` com `orderId` e código de erro claro; o suporte não precisa filtrar o ruído de módulos não relacionados para achar a causa raiz.

---

## 10 — Falha independente

**Definição.** Falhas são **delimitadas**: timeouts, retries com backoff, bulkheads, circuit breaking, idempotência — para que a indisponibilidade de um módulo não **cascate às cegas**.

**Regras para agentes.**

- Chamadas entre módulos devem ter semântica de timeout e falha **explícita**; "pender para sempre" é um bug de design na fronteira.
- Handlers assíncronos devem ser **idempotentes** ou deduplicados onde duplicatas são possíveis.

**Exemplo abstrato.** Quando Recommendations está fora do ar, o Checkout ainda completa usando defaults ou um nível em cache; a UI degrada em vez de bloquear a compra.
