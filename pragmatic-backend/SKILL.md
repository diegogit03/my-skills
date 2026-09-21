---
name: pragmatic-nestjs
description: Orientação para construção de apps Backend com Nest.JS
---

# Pragmatic NestJS

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

Controllers -> Services -> Repository/Entity

Tipos de módulos:
- Infraestrutura
- Dominio

Entidades Ricas

Services = orquestradores

Separação Vertical > Separação Horizontal

## Referências

| Tópico        | Referência                          | Carregar Quando                                    |
| ------------- | ----------------------------------- | -------------------------------------------------- |
| Autenticação  | `references/authentication.md`      | Configurar auth: Personal Access Token, guards e decorators |
| Testes        | `references/testing-patterns.md`    | Escrever testes de serviço, controller, integração ou E2E  |
| Comunicação   | `references/module-communication.md`| Implementar eventos entre módulos, publishers e handlers   |
| Arquitetura   | `references/architecture-patterns.md` | Padrão de serviço simples, config strict do TypeScript    |
| Isolamento Estado | `references/state-isolation.md`   | Nomenclatura de entidades, detecção de duplicatas e anti-padrões |
