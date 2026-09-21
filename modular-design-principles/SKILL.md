---
name: modular-design-principles
description: >
  Orientação agnóstica de tecnologia para sistemas modulares: contextos delimitados, fronteiras claras,
  composabilidade, isolamento de estado, contratos explícitos, contenção de falhas, fluxos de trabalho de
  scaffolding, critérios de divisão/fusão, sub-unidades dentro de um contexto e sinais de revisão de
  conformidade. Use ao projetar ou revisar estrutura de módulos, fronteiras de serviços, layout de pacotes,
  dependências transversais, "como devemos dividir isso?", avaliações de modularidade, acoplamento entre
  domínios, design de contexto greenfield, ou discussões de arquitetura sem assumir um framework, linguagem
  ou layout de repositório específicos. NÃO use para executar o pipeline completo de decomposição de
  repositório dos Padrões 1–5 ou inventários por padrão (use modular-decomposition), roteiros de extração
  em fases como entregável principal (use decomposition-planning-roadmap), ou estratégia de migração de
  legado de ponta a ponta (use legacy-migration-planner).
---

# Princípios de Design Modular

Use esta skill ao raciocinar sobre **estrutura e fronteiras** em qualquer codebase. Ela intencionalmente evita nomes de frameworks, convenções de pastas e ferramentas — mapeie os princípios para sua stack localmente.

## O que carregar

| Tarefa | Onde |
|--------|------|
| Tabela de princípios + violações + fluxos de trabalho (este arquivo) | `SKILL.md` |
| Definição por princípio, regras para agentes, exemplos abstratos | `references/principles.md` |

---

## Modelo mental em camadas

- **Raízes de composição** (aplicações, hosts, runners): conectam os módulos; mantenha a orquestração enxuta.
- **Módulos / contextos delimitados**: unidades coesas de comportamento e propriedade de dados; cada um deve ser compreensível e testável por si só.
- **Kernels compartilhados** (use com parcimônia): apenas conceitos estáveis e verdadeiramente transversais; resista a transformá-los em um bau de "tudo que todos precisam".

Como você organiza isso fisicamente (mono repo, multi repo, pacotes, bibliotecas) é uma **escolha de entrega**, não a definição de modularidade. Os princípios abaixo continuam se aplicando.

---

## Os dez princípios

| # | Princípio | Intenção |
|---|-----------|----------|
| 1 | **Fronteiras bem definidas** | Uma **superfície pública** pequena e estável; todo o resto é interno. Consumidores dependem de contratos, não de internals. |
| 2 | **Composabilidade** | Módulos podem ser usados sozinhos ou combinados sem conhecimento especial dos internals uns dos outros. |
| 3 | **Independência** | Nenhum estado mutável compartilhado oculto entre fronteiras; cada módulo deve ser testável isoladamente (com fakes ou test doubles nas bordas). |
| 4 | **Escala individual** | Recursos (compute, armazenamento, rate limits, tamanho de batch) podem ser ajustados **por módulo** onde importa, sem reescrever os outros. |
| 5 | **Comunicação explícita** | Interação entre módulos usa **contratos documentados** (APIs, eventos, mensagens, tipos compartilhados) — não acoplamento incidental. |
| 6 | **Substituibilidade** | Dependências de outros módulos são expressas por meio de **interfaces ou protocolos** para que as implementações possam mudar. |
| 7 | **Independência de implantação** | Módulos não assumem que compartilham processo, host ou cadência de release, a menos que isso seja uma decisão arquitetural explícita. |
| 8 | **Isolamento de estado** | Cada módulo **possui** seu estado persistente e nomenclatura; nenhum compartilhamento silencioso do mesmo datastore lógico ou nomes globais ambíguos entre fronteiras. |
| 9 | **Observabilidade** | Cada módulo pode ser diagnosticado por si só: logs, métricas, traces, health — atribuíveis à unidade que os emitiu. |
| 10 | **Falha independente** | Falhas são **contidas** (timeouts, bulkheads, circuit breaking, idempotência) para que a indisponibilidade de um módulo não cascata às cegas. |

O **Princípio 8** costuma ser o mais difícil: propriedade ambígua de dados ou nomes é uma fonte frequente de bugs de integração que "funcionam até não funcionarem".

Para **profundidade** (regras para agentes + exemplos abstratos por princípio), carregue `references/principles.md`.

---

## Violações típicas (descritas de forma abstrata)

1. **Conceitos colidentes** — o mesmo nome ou schema para coisas diferentes em módulos diferentes, ou definições "globais" duplicadas que divergem com o tempo.
2. **Persistência por alcançamento (reach-through)** — um módulo lendo ou escrevendo tabelas, buckets ou documentos de outro módulo **sem** passar por um contrato acordado.
3. **Propriedade de dados centralizada** — uma única camada de persistência que registra e expõe **todos** os stores de **todos** os módulos, encorajando acoplamento oculto.
4. **Lógica na borda** — regras de negócio em adaptadores de transporte (handlers HTTP, UI, CLI) em vez de código de domínio/aplicação.
5. **Borda falando direto com o armazenamento** — adaptadores dependendo de APIs de persistência de baixo nível em vez de casos de uso ou serviços de aplicação.
6. **Transações sem escopo** — escritas que cruzam fronteiras sem propriedade clara de transação e semântica de falha.
7. **Exports com vazamento** — repositórios, serviços internos ou tipos de implementação expostos como API pública do módulo.
8. **Facades que não são enxutas** — pontos de entrada "públicos" que embutem consulta, mapeamento ou política em vez de delegar para a camada correta dentro do módulo.

---

## Criando um contexto delimitado (fluxo de trabalho)

Use ao introduzir uma área **nova** e coesa do sistema (módulo greenfield ou domínio extraído).

1. **Escopo e linguagem** — Nomeie o contexto; liste substantivos/verbos centrais (**linguagem ubíqua**). Rejeite nomes vagos que colidam com outros contextos.
2. **Responsabilidades** — Quais decisões acontecem **apenas** aqui? O que está explicitamente *fora* do escopo?
3. **Propriedade de estado** — Quais fatos são **autoritativos** neste contexto? Onde são armazenados conceitualmente (mesmo que a tecnologia de armazenamento não esteja decidida)?
4. **Contrato público** — Operações e/ou eventos que outros contextos podem usar. Versione ou evolua esse contrato intencionalmente.
5. **Integrações** — Para cada vizinho: chamada síncrona, mensagem assíncrona, read model compartilhado ou sync em batch? Documente **consistência** (imediata, eventual) e comportamento de **falha**.
6. **Invariantes e ciclos de vida** — O que sempre deve ser verdadeiro dentro desta fronteira? O que inicia/completa um ciclo de vida?
7. **Verificação de isolamento** — Você consegue testar o comportamento central **sem** subir contextos não relacionados (fakes nas portas)?
8. **Observabilidade** — Como você rastreará uma requisição ou job por **este** contexto com identificadores claros?

**Interação entre módulos** (durante o design): prefira o contrato **mínimo**; defina **timeouts**, **retries**, **idempotência** para fluxos assíncronos; evite acesso direto "temporário" a stores como atalho.

---

## Quando dividir ou fundir

**Padrão:** **menos fronteiras** até que dor real apareça — "flat is often better" do que fragmentação prematura. Dividir adiciona custo de coordenação, versionamento e operação.

### Teste dos seis critérios (favoreça dividir quando vários forem verdadeiros)

| # | Critério | Pergunta |
|---|----------|----------|
| 1 | **Linguagem** | As sub-áreas usam **vocabulários diferentes** ou definições conflitantes da mesma palavra? |
| 2 | **Ritmo de mudança** | As partes **mudam em cadências diferentes** ou por razões não relacionadas (a maioria das edições toca apenas um lado)? |
| 3 | **Escala / SLO** | As partes precisam de metas **diferentes** de throughput, latência ou disponibilidade? |
| 4 | **Consistência** | Elas precisam de **fronteiras de transação diferentes** (não conseguem compartilhar limparamente um modelo de escrita atômico)? |
| 5 | **Propriedade** | **Equipes diferentes** ou linhas de propriedade claras reduziriam conflito e churn de review? |
| 6 | **Sinal de dor** | Há dor de integração **observável**: efeitos cascata, medo de mudar, incerteza sobre quem é dono de um bug? |

**Coesão / acoplamento (qualitativo).** Favoreça **alta coesão** dentro de um módulo e **acoplamento baixo e explícito** entre módulos. Se a única motivação é "os arquivos ficaram grandes" ou "estética de pastas", **funda ou espere**.

### Quando fundir ou ainda não dividir

- As fronteiras são **artificiais** (mesma linguagem, mesmo ciclo de vida, chamadas cruzadas constantes).
- Dividir **duplicaria** lógica ou dados sem uma regra clara de **escritor único**.
- A equipe não está pronta para **possuir** contratos, versionamento e operação de unidades extras.

### Prompts de decisão (curtos)

- A separação **reduziria** o acoplamento acidental mais do que **aumenta** o custo de coordenação?
- Existe uma fronteira natural de **linguagem ubíqua**, ou apenas uma emenda técnica?

---

## Sub-unidades dentro de um contexto delimitado

Às vezes uma fronteira externa está correta, mas dentro dela há **sub-áreas nomeadas** (subdomínios, áreas de funcionalidades). Os princípios ainda se aplicam **dentro** do contexto.

**Propriedade**

- Cada sub-unidade deve **possuir** sua fatia do modelo e das preocupações de persistência quando possível — evite uma única mega camada de registro que conecta **todos** os stores e repositórios de **todas** as sub-unidades em um só lugar (encoraja reach-through e acoplamento oculto).

**Acesso entre sub-unidades**

- Prefira **APIs de aplicação internas** ou **facades internas enxutas** (mesmo contexto, superfície explícita) a pares importando diretamente os tipos de armazenamento uns dos outros.
- Para fluxos assíncronos, prefira **payloads enriquecidos** para que handlers não precisem **conversar** entre sub-unidades para obter dados que poderiam viajar com o evento/comando.

**Kernel compartilhado dentro do contexto**

- Tipos ou enums compartilhados, pequenos e estáveis podem viver em uma **área compartilhada estreita** — mas resista a um depósito crescente de "utils" que se torna o verdadeiro ponto de acoplamento.

**Anti-padrão:** Um único sub-módulo de "persistência" ou "dados" que se torna o **único** lugar que conhece todas as tabelas/documentos de todas as sub-unidades, e todos os outros alcançam através dele — os mesmos problemas de reach-through entre contextos, mas **dentro** da fronteira.

---

## Passada de conformidade de arquitetura

Use para **reviews** ou **audits** sem assumir ferramentas. Trate os itens como **sinais**, não prova — confirme com especialistas do domínio.

### Sinais de dependência e API

- **Entrada vs saída:** Dependências devem alinhar-se com sua arquitetura escolhida (ex.: domínio no centro, adaptadores fora). Vazamentos **para dentro** de tipos de infraestrutura na lógica central são um smell.
- **Superfície pública:** Você consegue listar operações/eventos/tipos **exportados** sem incluir armazenamento ou serviços internos? Se não, as fronteiras estão com vazamento.
- **Imports entre vizinhos:** Tipos ou clients do **módulo A** usados no **módulo B** — são apenas tipos de **contrato**, ou tipos de persistência/implementação?

### Sinais de persistência e dados

- **Reach-through:** Referências aos dados **físicos** de outro contexto (schema, collection, nome de bucket) fora de um contrato acordado.
- **Colisões de nomenclatura:** Mesmo nome lógico para coisas diferentes, ou IDs globais compartilhados sem uma regra de mapeamento documentada.
- **Propriedade de transações:** Escritas que cruzam contextos sem uma regra clara de **saga**, **outbox** ou **dono único** e casos de falha documentados.

### Sinais operacionais

- **Culpa:** Incidentes em que "não sabemos qual módulo é dono desta linha/comportamento" → lacuna de propriedade ou observabilidade.
- **Cascatas:** A lentidão ou falha de uma dependência derruba jornadas de usuário não relacionadas → faltam **timeouts**, **bulkheads** ou caminhos de **degradação**.

### Heurística de severidade (para relatórios)

| Nível | Significado |
|-------|-------------|
| **P0** | Risco de corrupção de dados, violação de fronteira de segurança, ou persistência entre contextos sem contrato |
| **P1** | Propriedade pouco clara, API pública com vazamento, semântica de falha ausente nas fronteiras |
| **P2** | Lacunas de observabilidade, smells de composabilidade, dívida técnica que aumenta acoplamento futuro |

**Nota de maturidade:** A pontuação é **qualitativa** a menos que a equipe defina gates numéricos. Use tendências: menos P0/P1 ao longo do tempo, contratos mais claros.

---

## Checklist rápido (antes de propor estrutura)

- [ ] A API pública é mínima; internals não são exportados de forma casual.
- [ ] Nomes e propriedade de armazenamento são inequívocos por módulo.
- [ ] Nenhum atalho de persistência entre módulos sem um contrato explícito.
- [ ] Regras de negócio ficam atrás de uma camada de aplicação/domínio clara, não apenas em adaptadores.
- [ ] Chamadas entre módulos têm comportamento explícito de falha e timeout.
- [ ] A observabilidade consegue responder "qual módulo falhou e por quê?" sem escavação.
- [ ] Se o contexto tem sub-unidades: cada uma tem propriedade clara; nenhum bau monolítico de persistência que "registra tudo".

---

## Relação com skills específicas de stack

Quando um projeto tem **convenções concretas** (módulos de framework, DI, padrões de repositório, layout de pastas, codegen, checks de CI), prefira esses documentos para o **como** implementar. Use **esta** skill para o **porquê** das fronteiras existirem e **o que** o bom design modular otimiza — para que a orientação específica de stack permaneça alinhada aos mesmos princípios.
