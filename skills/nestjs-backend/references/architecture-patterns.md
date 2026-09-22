# Padrões de Arquitetura de Módulo

Módulos seguem **Feature Folders** organizados por **flat-by-aggregate**: 1 conceito de negócio (agregado) = 1 pasta, com as camadas técnicas como sufixos de arquivo — não como pastas. Para os componentes fundamentais (serviços, entidades e repositórios), ver `references/building-blocks.md`.

## Sumário

1. Feature Folders — Flat-by-Aggregate (linha ~10)
2. Regras do layout flat (linha ~45)

---

## 1. Feature Folders — Flat-by-Aggregate

Organização flat-by-aggregate: `ls module/` revela o domínio (Screaming Architecture), não o framework.

```
finance/
  wallets/
    wallet.entity.ts
    wallet.repository.ts
    wallet.service.ts
    wallet.controller.ts
    wallet.dto.ts
    wallet.admin.controller.ts        # painel admin (BFF por sufixo)
    wallet.admin.dto.ts
    wallet.dashboard.controller.ts    # painel dashboard (BFF por sufixo)
    wallet.dashboard.dto.ts
    __tests__/
      wallet.e2e-spec.ts
  transactions/
    transaction.entity.ts
    transaction.repository.ts
    transaction.service.ts
    transaction.controller.ts
    transaction.dto.ts
    __tests__/
      transaction.e2e-spec.ts
  migrations/                # Migrações do módulo
  finance.module.ts
```

---

## 2. Regras do layout flat

- **1 agregado = 1 pasta.** Todo o código de produção do agregado vive junto; camadas técnicas viram sufixos (`.entity.ts`, `.repository.ts`, `.service.ts`, `.controller.ts`).
- **Profundidade ≤ 2.** Nunca pastas de camada técnica (`core/`, `http/`, `persistence/`).
- **Regra de dependência mantida por sufixo:** controller → service → entity/repositório. As dependências apontam para o domínio, só que expressas por co-localização.
- **E2E por agregado** em `<aggregate>/__tests__/` (ver `references/testing-patterns.md`).
- **Service como unidade default:** agregado com muitos arquivos → quebre em sub-agregados dentro do mesmo módulo.
- **Múltiplos consumidores (BFF por sufixo):** quando o agregado atende painéis distintos (admin, dashboard, mobile), cada painel vira `<aggregate>.<painel>.controller.ts` + `<aggregate>.<painel>.dto.ts` dentro da pasta do agregado — nunca uma pasta de slice. DTO por painel (payload, validação e forma de resposta variam por consumidor); a regra, não. Regra própria do painel → `<aggregate>.<painel>.service.ts`, *orquestrando* os services do agregado (composição), nunca reimplementando. Painéis com agregados e modelos de autorização totalmente distintos → módulos separados, não camadas internas.
