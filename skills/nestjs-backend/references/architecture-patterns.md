# Padrões de Arquitetura de Módulo

Módulos seguem **Feature Folders** organizados por **flat-by-aggregate**: 1 conceito de negócio (agregado) = 1 pasta, com as camadas técnicas como sufixos de arquivo — não como pastas. Para os componentes fundamentais (serviços, entidades e repositórios), ver `references/building-blocks.md`.

## Sumário

1. Feature Folders — Flat-by-Aggregate (linha ~10)
2. Regras do layout flat (linha ~40)
3. Quando separar em subdomínio (linha ~60)

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

## 3. Quando separar em subdomínio

Use estrutura por subdomínio (profundidade 3: `<module>/<subdomain>/<aggregate>/`) apenas quando **4+ de 6** critérios se aplicam:

1. Personas de usuário diferentes (admin vs cliente)?
2. Modelos de autorização diferentes?
3. Modelos de execução diferentes (REST vs fila vs GraphQL)?
4. Características de escala diferentes (leitura vs escrita, CPU vs I/O)?
5. Poderia ser implantado independentemente?
6. Pode falhar isoladamente?

**Default: flat.** Red flags para NÃO dividir: "parece grande demais", "para facilitar achar código" (resolva com nomes de agregado, não pastas de camada), features fortemente acopladas, espelhar o organograma.

### Exemplo: módulo `content` com subdomínios

Um módulo de conteúdo que gerencia tanto o catálogo (área pública, leitura intensiva, cache agressivo) quanto o gerenciamento editorial (área admin, escrita, autorização própria) atende aos critérios de persona, autorização, escala e execução. Em vez de um `content/` flat com 10+ agregados misturados, organize por subdomínio:

```
content/
  catalog/                        # subdomínio: leitura pública
    products/
      product.entity.ts
      product.repository.ts
      product.service.ts
      product.controller.ts
      __tests__/
        product.e2e-spec.ts
    categories/
      category.entity.ts
      category.repository.ts
      category.service.ts
      category.controller.ts
      __tests__/
    catalog.module.ts             # registra providers do subdomínio
  editorial/                      # subdomínio: gestão (admin)
    drafts/
      draft.entity.ts
      draft.repository.ts
      draft.service.ts
      draft.controller.ts
      __tests__/
    publishing/
      publish.entity.ts
      publish.repository.ts
      publish.service.ts
      publish.controller.ts
      __tests__/
    editorial.module.ts
  migrations/
  content.module.ts               # compõe os subdomínios
```

**Regras entre subdomínios:**

- Cada subdomínio possui seus repositórios e registra seus providers no próprio `<subdomain>.module.ts`.
- O módulo raiz (`content.module.ts`) apenas importa os módulos dos subdomínios.
- Leitura entre subdomínios **não** acessa repositório alheio — exponha o que for necessário via service público do subdomínio (ou facade de delegação) e consuma pelo import `@modules/content`.

Contra-exemplo: `billing` com `invoices/` + `payments/` + `refunds/` que compartilham a mesma transação ACID — acoplamento alto indica que pertencem ao mesmo subdomínio; mantenha flat.
