# Padrões de Arquitetura de Módulo

Cada módulo usa **um** destes padrões — nunca misture os dois dentro do mesmo módulo. Para os componentes fundamentais (serviços, entidades e repositórios), ver `references/building-blocks.md`.

## Sumário

1. Layer Architecture (linha ~13)
2. Feature Folders (linha ~35)
3. Como Escolher (linha ~55)

---

## 1. Layer Architecture

Padrão para módulos com domínio complexo.

```
finance/
  core/
    service/               # Serviços (orquestradores)
    entity/                # Entidades TypeORM (entidades de domínio)
  http/
    controllers/           # Controllers, DTOs
  persistence/
    migrations/            # Migrações do módulo
    repository/            # Repositórios (classes concretas)
  finance.module.ts
```

---

## 2. Feature Folders

Padrão para módulos simples ou majoritariamente CRUD. Cada feature é autocontida com service + controller lado a lado.

```
finance/
  wallets/
    wallet.service.ts
    wallet.controller.ts
  transactions/
    transaction.service.ts
    transaction.controller.ts
  finance.module.ts
```

---

## 3. Como Escolher

| Critério                                  | Layer Architecture | Feature Folders |
| ----------------------------------------- | ------------------ | --------------- |
| Regras de negócio ricas, invariantes      | ✅                 | ❌              |
| Múltiplas entidades relacionadas          | ✅                 | ❌              |
| Majoritariamente CRUD                     | ❌ over-engineering| ✅              |
| Features independentes e simples          | ❌ over-engineering| ✅              |
| Serviços com orquestração complexa        | ✅                 | ⚠️ se crescer, migre |

**Regra prática:** comece com Feature Folders; quando um módulo acumular regras de negócio complexas, múltiplas entidades e necessidade de testar serviços isolados, migre para Layer Architecture.
