---
name: nestjs-backend
description: Orientação para construção de APIs Backend com Nest.JS
---

# Pragmatic NestJS

## Estrutura do Projeto

```
src/
  common/                  # Shared kernel — nada de lógica de negócio aqui
    contracts/             # Contratos entre módulos (eventos, interfaces)
    infrastructure/        # Publishers, guards, decorators, testing utils
  modules/                 # Módulos de domínio
    finance/
    identity/
  app.module.ts
  main.ts
test/                      # Testes E2E
```

Aliase de import (sempre use estes paths, nunca caminhos relativos entre módulos):

- `@common/*` → `src/common/*`
- `@modules/<nome>` → `src/modules/<nome>`

## Padrões de Arquitetura de Módulo

Módulos seguem **Feature Folders com flat-by-aggregate** (ver `references/architecture-patterns.md`):

```
finance/
  wallets/                   # 1 agregado = 1 pasta
    wallet.entity.ts
    wallet.repository.ts
    wallet.service.ts
    wallet.controller.ts
    __tests__/
  transactions/
    transaction.entity.ts
    ...
  migrations/                # Migrações do módulo
  finance.module.ts
```

## Princípios Fundamentais

**10 Princípios de Monolito Modular** — estes sobrepõem os defaults gerais de NestJS quando conflitam:

1. **Fronteiras**: Interfaces claras entre módulos, acoplamento mínimo
2. **Composabilidade**: Módulos podem ser recombinados dinamicamente
3. **Independência**: Cada módulo é autocontido com seu próprio domínio
4. **Escalabilidade**: Otimização por módulo sem mudanças sistêmicas
5. **Comunicação Explícita**: Contratos entre módulos, nunca implícitos
6. **Substituibilidade**: Qualquer módulo pode ser substituído sem impacto no sistema
7. **Separação Lógica de Implantação**: Mesmo em monolito, manter separação
8. **Isolamento de Estado**: Fronteiras de dados estritas — nenhuma tabela de banco compartilhada
9. **Observabilidade**: Monitoramento e tracing a nível de módulo
10. **Resiliência**: Falhas em um módulo não cascam

## Referências

| Tópico        | Referência                          | Carregar Quando                                    |
| ------------- | ----------------------------------- | -------------------------------------------------- |
| Autenticação  | `references/authentication.md`      | Configurar auth: Personal Access Token, guards e decorators |
| Testes        | `references/testing-patterns.md`    | Escrever testes de entidade, serviço ou E2E (Vitest) |
| Comunicação   | `references/module-communication.md`| Eventos, publishers, handlers ou chamadas síncronas entre módulos (public API)   |
| Arquitetura   | `references/architecture-patterns.md` | Layer architecture e feature folders: estrutura, escolha do padrão e registro do módulo |
| Building Blocks | `references/building-blocks.md`     | Serviços, entidades TypeORM e repositórios (incl. base repository) |
