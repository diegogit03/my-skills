# Isolamento de Estado

## Sumário

1. Convenções de Nomenclatura de Entidades (linha ~10)
2. Entidades TypeORM e Isolamento de Tabelas (linha ~55)
3. Detecção de Entidades Duplicadas (linha ~130)
4. Detecção de Anti-Padrões (linha ~165)
5. Hook de Pre-Commit (linha ~205)

---

## 1. Convenções de Nomenclatura de Entidades

**Regra crítica:** Toda entidade DEVE ser prefixada com o nome do seu módulo. Nomes genéricos como `User`, `Plan`, `Item` causam colisões entre módulos e tornam impossível saber qual módulo é dono dos dados.

### Nomenclatura Correta de Entidades

| Módulo   | Nome da Entidade      | Tabela do Banco         |
| -------- | --------------------- | ----------------------- |
| Identity | `IdentityUser`        | `identity_users`        |
| Identity | `IdentityProfile`     | `identity_profiles`     |
| Billing  | `BillingPlan`         | `billing_plans`         |
| Billing  | `BillingSubscription` | `billing_subscriptions` |
| Orders   | `OrderRecord`         | `order_records`         |
| Content  | `ContentArticle`      | `content_articles`      |

### Nomenclatura Incorreta de Entidades

| ❌ Nome   | Problema                                   |
| --------- | ------------------------------------------ |
| `User`    | Qual módulo? Identity? Billing?            |
| `Plan`    | Plano de cobrança? Plano de assinatura?    |
| `Item`    | Item de pedido? Item de carrinho? Item de inventário? |
| `Profile` | Perfil de usuário? Perfil de empresa?      |

---

## 2. Entidades TypeORM e Isolamento de Tabelas

Cada módulo registra suas próprias entidades via `TypeOrmModule.forFeature`. O nome da tabela (`@Entity`) segue o prefixo do módulo.

```typescript
// ✅ Correto: Entidade prefixada pelo módulo com tabela explícita
// libs/identity/infrastructure/entities/user.ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, UpdateDateColumn } from 'typeorm'

@Entity('identity_users')
export class IdentityUser {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column({ unique: true })
  email: string

  @Column()
  name: string

  @CreateDateColumn({ type: 'timestamptz' })
  createdAt: Date

  @UpdateDateColumn({ type: 'timestamptz' })
  updatedAt: Date
}

// libs/billing/infrastructure/entities/plan.ts
@Entity('billing_plans')
export class BillingPlan {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column()
  name: string

  @Column({ type: 'integer' })
  priceInCents: number

  @Column({ type: 'enum', enum: BillingInterval })
  interval: BillingInterval
}

// ❌ Errado: Nome genérico sem prefixo do módulo
@Entity('users') // Qual módulo é dono disso?
export class User {
  /* ... */
}
```

### Referências entre Módulos

**Regra-chave:** Referências entre módulos usam coluna de ID (ex.: `userId`), NÃO relações de chave estrangeira entre entidades de módulos diferentes. Relações (`@ManyToOne`, `@OneToMany`) devem existir apenas DENTRO de um módulo.

```typescript
// ✅ Correto: Billing referencia o usuário apenas por ID
// libs/billing/infrastructure/entities/subscription.ts
@Entity('billing_subscriptions')
export class BillingSubscription {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column()
  userId: string // Referência apenas por ID, sem FK para a tabela do módulo Identity

  @Column()
  planId: string

  @Column({ type: 'enum', enum: BillingSubscriptionStatus })
  status: BillingSubscriptionStatus

  @ManyToOne(() => BillingPlan)
  @JoinColumn({ name: 'planId' })
  plan: BillingPlan // Relação DENTRO do módulo Billing — permitida

  @CreateDateColumn({ type: 'timestamptz' })
  createdAt: Date
}

// ❌ Errado: FK cruzando a fronteira do módulo
@ManyToOne(() => IdentityUser) // Importa entidade de outro módulo
@JoinColumn({ name: 'userId' })
user: IdentityUser
```

### Registro por Módulo

Cada módulo registra apenas suas entidades — nenhuma camada central registra todas:

```typescript
// libs/identity/identity.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([IdentityUser, IdentityProfile])],
  /* ... */
})
export class IdentityModule {}
```

---

## 3. Detecção de Entidades Duplicadas

Rode estas verificações antes de cada commit ou merge de PR.

### Encontrar Nomes de Classes de Entidade Duplicados

```bash
# Encontrar classes @Entity com nomes duplicados entre módulos
grep -rn -A2 "@Entity" libs/ | grep "export class" | awk '{print $3}' | sed 's/ {//' | sort | uniq -d
```

### Encontrar Mapeamentos de Tabela Duplicados

```bash
# Encontrar nomes de tabela @Entity('...') duplicados
grep -r "@Entity(" libs/ | grep -o "'[^']*'" | sort | uniq -d
```

### Encontrar Relações de Chave Estrangeira entre Módulos

```bash
# Detectar imports de entidades de outros módulos em arquivos de entidades (FK cruzando módulos)
grep -rn "import.*entities" libs/*/infrastructure/entities/ | grep -v "libs/$(basename $(dirname))" | grep "shared" --invert
# Alternativa direta: listar todos os imports de entidades e revisar manualmente
grep -rn "from '@project/" libs/*/infrastructure/entities/ 2>/dev/null
```

---

## 4. Detecção de Anti-Padrões

### Detectar Estado Mutável Compartilhado

```bash
# Singletons mutáveis exportados — não devem existir
grep -r "export.*=.*new" libs/ | grep -v test | grep -v node_modules
```

### Detectar Imports Diretos entre Módulos

```bash
# Módulos devem importar apenas de @project/[module] (barrel index), nunca de caminhos profundos
grep -rn "from '@project/" libs/ | grep -v "/index" | grep -v "shared" | grep -v node_modules | grep -v ".spec."
```

### Detectar Chamadas Síncronas entre Módulos

```bash
# Chamadas diretas de serviços entre módulos — devem usar eventos
grep -rn "await.*Service\." libs/ | grep -v "this\." | grep -v test | grep -v node_modules
```

### Detectar Prefixos de Módulo Ausentes em Entidades

```bash
# Entidades com nomes genéricos de uma palavra (possíveis violações)
grep -rn "^export class [A-Z][a-z]*\b " libs/*/infrastructure/entities/ | grep -v "Error\|Exception\|Event\|Command\|Query\|Handler\|Dto\|Module\|Guard\|Filter"
```

### Detectar forFeature Centralizado

```bash
# Nenhuma camada central deve registrar entidades de todos os módulos
grep -rn "forFeature" apps/ | grep -v node_modules
```

---

## 5. Hook de Pre-Commit

Adicione isto ao `.husky/pre-commit` ou equivalente. Para uma versão mais abrangente, use `scripts/validate-isolation.sh` incluído nesta skill.

```bash
#!/bin/bash
echo "🔍 Executando verificações de isolamento de estado..."

ERRORS=0

# Verificação 1: Nomes de tabela @Entity duplicados
MAP_DUPES=$(grep -r "@Entity(" libs/ 2>/dev/null | grep -o "'[^']*'" | sort | uniq -d)
if [ -n "$MAP_DUPES" ]; then
  echo "❌ Mapeamentos de tabela duplicados encontrados: $MAP_DUPES"
  echo "   Correção: Prefixe cada tabela com o nome do seu módulo (ex.: billing_plans)"
  ERRORS=$((ERRORS + 1))
fi

# Verificação 2: Classes de entidade duplicadas entre módulos
CLASS_DUPES=$(grep -rn -A2 "@Entity" libs/ 2>/dev/null | grep "export class" | awk '{print $3}' | sed 's/ {//' | sort | uniq -d)
if [ -n "$CLASS_DUPES" ]; then
  echo "❌ Nomes de classes de entidade duplicados: $CLASS_DUPES"
  echo "   Correção: Prefixe cada classe com o nome do seu módulo (ex.: BillingPlan)"
  ERRORS=$((ERRORS + 1))
fi

# Verificação 3: Imports diretos entre módulos (não-barrel)
CROSS_IMPORTS=$(grep -rn "from '@project/" libs/ 2>/dev/null | grep -v "/index" | grep -v "shared" | grep -v node_modules | grep -v ".spec.")
if [ -n "$CROSS_IMPORTS" ]; then
  echo "⚠️  Imports diretos entre módulos detectados:"
  echo "$CROSS_IMPORTS"
  echo "   Correção: Importe apenas do barrel do módulo (@project/module-name)"
  ERRORS=$((ERRORS + 1))
fi

if [ $ERRORS -gt 0 ]; then
  echo "❌ Verificação de isolamento de estado falhou com $ERRORS erro(s). Corrija os problemas antes de commitar."
  exit 1
fi

echo "✅ Verificações de isolamento de estado passaram."
```
