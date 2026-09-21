# Isolamento de Estado

## Sumário

1. Convenções de Nomenclatura de Entidades (linha ~10)
2. Isolamento de Schema Prisma (linha ~50)
3. Detecção de Entidades Duplicadas (linha ~110)
4. Detecção de Anti-Padrões (linha ~150)
5. Hook de Pre-Commit (linha ~190)

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
| Orders   | `OrderItem`           | `order_items`           |
| Content  | `ContentArticle`      | `content_articles`      |

### Nomenclatura Incorreta de Entidades

| ❌ Nome   | Problema                                   |
| --------- | ------------------------------------------ |
| `User`    | Qual módulo? Identity? Billing?            |
| `Plan`    | Plano de cobrança? Plano de assinatura?    |
| `Item`    | Item de pedido? Item de carrinho? Item de inventário? |
| `Profile` | Perfil de usuário? Perfil de empresa?      |

### Nomenclatura de Models Prisma

```prisma
// ✅ Correto: Prefixado pelo módulo com mapeamento explícito de tabela
model IdentityUser {
  id    String @id @default(cuid())
  email String @unique
  name  String
  // ...
  @@map("identity_users")
}

model BillingPlan {
  id           String @id @default(cuid())
  name         String
  priceInCents Int
  // ...
  @@map("billing_plans")
}

// ❌ Errado: Nomes genéricos sem prefixo do módulo
model User {
  // ...
  @@map("users")  // Qual módulo é dono disso?
}
```

---

## 2. Isolamento de Schema Prisma

### Opção A: Schema Único, Prefixos por Módulo

Todos os models vivem em um único `schema.prisma`, mas são claramente prefixados e agrupados.

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ═══════════════════════════════════════════
// Módulo Identity
// ═══════════════════════════════════════════

model IdentityUser {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  @@map("identity_users")
}

// ═══════════════════════════════════════════
// Módulo Billing
// ═══════════════════════════════════════════

model BillingPlan {
  id            String              @id @default(cuid())
  name          String
  priceInCents  Int
  interval      BillingInterval
  subscriptions BillingSubscription[]
  @@map("billing_plans")
}

model BillingSubscription {
  id        String   @id @default(cuid())
  userId    String   // Referência apenas por ID, sem FK para IdentityUser
  planId    String
  status    BillingSubscriptionStatus
  plan      BillingPlan @relation(fields: [planId], references: [id])
  createdAt DateTime @default(now())
  @@map("billing_subscriptions")
}

enum BillingInterval {
  MONTHLY
  YEARLY
}

enum BillingSubscriptionStatus {
  ACTIVE
  CANCELLED
  PAST_DUE
}

// ═══════════════════════════════════════════
// Módulo Orders
// ═══════════════════════════════════════════

model OrderRecord {
  id        String      @id @default(cuid())
  userId    String      // Referência apenas por ID
  status    OrderStatus
  total     Decimal     @db.Decimal(10, 2)
  items     OrderItem[]
  createdAt DateTime    @default(now())
  @@map("order_records")
}

model OrderItem {
  id        String  @id @default(cuid())
  orderId   String
  productId String
  quantity  Int
  price     Decimal @db.Decimal(10, 2)
  order     OrderRecord @relation(fields: [orderId], references: [id])
  @@map("order_items")
}

enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}
```

**Regra-chave:** Referências entre módulos usam `userId String` (apenas o ID), NÃO `user IdentityUser @relation(...)`. Relações de chave estrangeira devem existir apenas DENTRO de um módulo.

### Opção B: Multi-Schema (Prisma 5.15+)

Para projetos maiores, use o suporte multi-schema do Prisma:

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  schemas  = ["identity", "billing", "orders"]
}

model IdentityUser {
  id    String @id @default(cuid())
  email String @unique
  @@schema("identity")
  @@map("users")
}

model BillingPlan {
  id   String @id @default(cuid())
  name String
  @@schema("billing")
  @@map("plans")
}
```

---

## 3. Detecção de Entidades Duplicadas

Rode estas verificações antes de cada commit ou merge de PR.

### Encontrar Nomes de Models Duplicados

```bash
# Para schemas Prisma — encontrar nomes de models duplicados
grep -r "^model " prisma/ | awk '{print $2}' | sort | uniq -d
```

### Encontrar Mapeamentos de Tabela Duplicados

```bash
# Encontrar valores @@map duplicados
grep -r '@@map(' prisma/ | grep -o '"[^"]*"' | sort | uniq -d
```

### Encontrar Relações de Chave Estrangeira entre Módulos

```bash
# Estas NÃO devem existir — apenas relações dentro do módulo são permitidas
grep -rn '@relation' prisma/schema.prisma | while read line; do
  echo "CHECK: $line"
  echo "  → Verifique se esta relação está DENTRO de um único módulo"
done
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
# Entidades/models com nomes genéricos de uma palavra (possíveis violações)
grep -rn "^export class [A-Z][a-z]*\b " libs/*/domain/ | grep -v "Error\|Exception\|Event\|Command\|Query\|Handler\|Dto\|Module\|Guard\|Filter"
```

---

## 5. Hook de Pre-Commit

Adicione isto ao `.husky/pre-commit` ou equivalente. Para uma versão mais abrangente, use o `scripts/validate-isolation.sh` incluído nesta skill.

```bash
#!/bin/bash
echo "🔍 Executando verificações de isolamento de estado..."

ERRORS=0

# Verificação 1: Nomes de models Prisma duplicados
DUPES=$(grep -r "^model " prisma/ 2>/dev/null | awk '{print $2}' | sort | uniq -d)
if [ -n "$DUPES" ]; then
  echo "❌ Nomes de models duplicados encontrados: $DUPES"
  echo "   Correção: Prefixe cada model com o nome do seu módulo (ex.: BillingPlan)"
  ERRORS=$((ERRORS + 1))
fi

# Verificação 2: Mapeamentos de tabela duplicados
MAP_DUPES=$(grep -r '@@map(' prisma/ 2>/dev/null | grep -o '"[^"]*"' | sort | uniq -d)
if [ -n "$MAP_DUPES" ]; then
  echo "❌ Mapeamentos de tabela duplicados encontrados: $MAP_DUPES"
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
