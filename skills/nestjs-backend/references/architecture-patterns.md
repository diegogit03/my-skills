# Padrões de Arquitetura de Módulo

Módulos seguem **Feature Folders** organizados por **flat-by-aggregate**: 1 conceito de negócio (agregado) = 1 pasta, com as camadas técnicas como sufixos de arquivo — não como pastas. Para os componentes fundamentais (serviços, entidades e repositórios), ver `references/building-blocks.md`.

## Sumário

1. Feature Folders — Flat-by-Aggregate (linha ~10)
2. Regras do layout flat (linha ~40)
3. Módulo com múltiplos painéis (BFF) (linha ~50)

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

## 3. Módulo com múltiplos painéis (BFF)

Quando o módulo atende múltiplos consumidores (admin, dashboard, mobile), **não** duplique regras nem volte para pastas de camada técnica. Os agregados carregam as regras compartilhadas; cada painel vira uma pasta de slice com **apenas controller + DTO**, delegando para os services do agregado.

```
finance/
  wallets/                          # núcleo compartilhado (regras de negócio)
    wallet.entity.ts
    wallet.repository.ts
    wallet.service.ts
    __tests__/
  transactions/
    transaction.entity.ts
    transaction.repository.ts
    transaction.service.ts
    __tests__/
  admin/                            # slice de apresentação
    wallet.admin.controller.ts
    wallet.admin.dto.ts
  dashboard/
    wallet.dashboard.controller.ts
    wallet.dashboard.dto.ts
  mobile/
    wallet.mobile.controller.ts
    wallet.mobile.dto.ts
  migrations/
  finance.module.ts
```

Regras:

- **Profundidade ≤ 2 se mantém** — `admin/` e `wallets/` estão no mesmo nível; nenhum voltou a ser pasta de camada técnica.
- **Regra de dependência intacta:** controller (painel) → service (agregado) → repository. O painel nunca acessa repository nem duplica regra de negócio.
- **DTO por painel** é a essência do BFF: payload, validação e forma de resposta variam por consumidor; a regra, não.
- **Regra própria do painel** (ex.: admin aprova com fluxo próprio) → crie `wallet.admin.service.ts` dentro do slice, *orquestrando* os services do agregado (composição), nunca reimplementando.

Se os painéis tiverem agregados totalmente distintos e modelos de autorização/execução diferentes, o problema é de **módulos separados**, não de camadas internas.
