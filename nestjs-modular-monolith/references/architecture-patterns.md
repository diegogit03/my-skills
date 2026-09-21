# Padrões de Arquitetura

## Sumário

1. Camadas da Clean Architecture (linha ~15)
2. Blocos de Construção do DDD (linha ~80)
3. Padrão CQRS — Opcional (linha ~140)
4. Padrão de Serviço Simples — Default (linha ~200)
5. Configuração do Workspace NX (linha ~240)
6. Configuração Strict do TypeScript (linha ~310)

---

## 1. Camadas da Clean Architecture

Cada módulo segue quatro camadas, com dependências apontando para dentro:

```
Presentation → Application → Domain ← Infrastructure
```

**Camada de Domínio** (mais interna — sem dependências externas):

```typescript
// libs/[module]/domain/entities/billing-plan.entity.ts
export class BillingPlan {
  constructor(
    public readonly id: string,
    public readonly name: string,
    public readonly priceInCents: number,
    public readonly interval: BillingInterval,
    public readonly createdAt: Date,
  ) {}

  isActive(): boolean {
    return this.priceInCents > 0
  }

  canUpgradeTo(target: BillingPlan): boolean {
    return target.priceInCents > this.priceInCents
  }
}

export enum BillingInterval {
  MONTHLY = 'MONTHLY',
  YEARLY = 'YEARLY',
}
```

**Camada de Domínio — Interface de Repositório:**

```typescript
// libs/[module]/domain/repositories/billing-plan.repository.ts
export interface BillingPlanRepository {
  findById(id: string): Promise<BillingPlan | null>
  findAll(): Promise<BillingPlan[]>
  save(plan: BillingPlan): Promise<BillingPlan>
  delete(id: string): Promise<void>
}

export const BILLING_PLAN_REPOSITORY = Symbol('BillingPlanRepository')
```

**Camada de Aplicação** (orquestra o domínio, define casos de uso):

```typescript
// libs/[module]/application/services/billing-plan.service.ts
@Injectable()
export class BillingPlanService {
  constructor(
    @Inject(BILLING_PLAN_REPOSITORY)
    private readonly repository: BillingPlanRepository,
    private readonly eventPublisher: EventPublisher,
  ) {}

  async create(name: string, priceInCents: number, interval: BillingInterval): Promise<BillingPlan> {
    const plan = new BillingPlan(generateId(), name, priceInCents, interval, new Date())
    const saved = await this.repository.save(plan)
    await this.eventPublisher.publish('billing.plan.created', { planId: saved.id, name: saved.name })
    return saved
  }

  async findById(id: string): Promise<BillingPlan> {
    const plan = await this.repository.findById(id)
    if (!plan) throw new BillingPlanNotFoundError(id)
    return plan
  }
}
```

**Camada de Infraestrutura** (implementa as interfaces do domínio):

```typescript
// libs/[module]/infrastructure/repositories/prisma-billing-plan.repository.ts
@Injectable()
export class PrismaBillingPlanRepository implements BillingPlanRepository {
  constructor(private readonly prisma: PrismaService) {}

  async findById(id: string): Promise<BillingPlan | null> {
    const data = await this.prisma.billingPlan.findUnique({ where: { id } })
    return data ? this.toDomain(data) : null
  }

  async save(plan: BillingPlan): Promise<BillingPlan> {
    const data = await this.prisma.billingPlan.upsert({
      where: { id: plan.id },
      update: { name: plan.name, priceInCents: plan.priceInCents },
      create: { id: plan.id, name: plan.name, priceInCents: plan.priceInCents, interval: plan.interval },
    })
    return this.toDomain(data)
  }

  async delete(id: string): Promise<void> {
    await this.prisma.billingPlan.delete({ where: { id } })
  }

  async findAll(): Promise<BillingPlan[]> {
    const data = await this.prisma.billingPlan.findMany()
    return data.map(this.toDomain)
  }

  private toDomain(data: PrismaBillingPlanRecord): BillingPlan {
    return new BillingPlan(data.id, data.name, data.priceInCents, data.interval as BillingInterval, data.createdAt)
  }
}
```

**Camada de Apresentação** (interface HTTP):

```typescript
// libs/[module]/presentation/billing-plan.controller.ts
@Controller('billing/plans')
@ApiTags('billing-plans')
export class BillingPlanController {
  constructor(private readonly billingPlanService: BillingPlanService) {}

  @Post()
  @ApiOperation({ summary: 'Criar plano de cobrança' })
  @ApiResponse({ status: 201, type: BillingPlanResponseDto })
  async create(@Body() dto: CreateBillingPlanDto): Promise<BillingPlanResponseDto> {
    const plan = await this.billingPlanService.create(dto.name, dto.priceInCents, dto.interval)
    return BillingPlanResponseDto.from(plan)
  }
}
```

---

## 2. Blocos de Construção do DDD

### Entidades

Objetos com identidade e ciclo de vida. Use nomes prefixados pelo módulo.

```typescript
// ✅ Correto: Entidade com prefixo do módulo
export class IdentityUser {
  constructor(
    public readonly id: string,
    public email: string,
    public name: string,
    private passwordHash: string,
  ) {}

  updateProfile(name: string): void {
    if (!name?.trim() || name.trim().length < 2) {
      throw new InvalidUserNameError('O nome deve ter pelo menos 2 caracteres')
    }
    this.name = name.trim()
  }

  verifyPassword(hasher: PasswordHasher, plainPassword: string): boolean {
    return hasher.verify(this.passwordHash, plainPassword)
  }
}

// ❌ Errado: Nome genérico sem prefixo do módulo
export class User {
  /* ... */
}
```

### Value Objects

Objetos imutáveis definidos por seus atributos, não por identidade.

```typescript
export class Money {
  constructor(
    public readonly amount: number,
    public readonly currency: string,
  ) {
    if (amount < 0) throw new Error('O valor não pode ser negativo')
    if (!currency || currency.length !== 3) throw new Error('Código de moeda inválido')
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) throw new Error('Moedas incompatíveis')
    return new Money(this.amount + other.amount, this.currency)
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency
  }
}
```

### Agregados

Cluster de entidades e value objects com uma entidade raiz. Acesso externo apenas pela raiz.

```typescript
export class OrderAggregate {
  private items: OrderItem[] = []

  constructor(
    public readonly id: string,
    public readonly customerId: string,
    private status: OrderStatus = OrderStatus.PENDING,
  ) {}

  addItem(productId: string, quantity: number, price: Money): void {
    if (this.status !== OrderStatus.PENDING) {
      throw new OrderNotModifiableError(this.id)
    }
    const existing = this.items.find((i) => i.productId === productId)
    if (existing) {
      existing.updateQuantity(existing.quantity + quantity)
    } else {
      this.items.push(new OrderItem(generateId(), productId, quantity, price))
    }
  }

  confirm(): void {
    if (this.items.length === 0) throw new EmptyOrderError(this.id)
    this.status = OrderStatus.CONFIRMED
  }

  get total(): Money {
    return this.items.reduce((sum, item) => sum.add(item.subtotal), new Money(0, 'USD'))
  }
}
```

### Exceções de Domínio

Erros específicos do domínio que mapeiam para códigos de status HTTP via exception filters.

```typescript
// libs/[module]/domain/exceptions/
export class DomainException extends Error {
  constructor(message: string) {
    super(message)
    this.name = this.constructor.name
  }
}

export class OrderNotFoundError extends DomainException {
  constructor(id: string) {
    super(`Pedido ${id} não encontrado`)
  }
}

export class OrderNotModifiableError extends DomainException {
  constructor(id: string) {
    super(`Pedido ${id} não pode ser modificado`)
  }
}

export class EmptyOrderError extends DomainException {
  constructor(id: string) {
    super(`Pedido ${id} não tem itens`)
  }
}
```

---

## 3. Padrão CQRS — Opcional

> ⚠️ **CQRS NÃO é o default.** Use-o apenas quando o domínio genuinamente se beneficia de separar modelos de leitura e escrita. Para a maioria dos módulos, o padrão de serviço simples (seção 4) é suficiente.

**Quando usar CQRS:**

- Padrões de leitura e escrita diferem significativamente
- Workloads com muitas leituras precisam de queries otimizadas
- Lógica de domínio complexa nas escritas, recuperação de dados simples nas leituras
- Necessidades de escala diferentes para leituras vs escritas

**Quando NÃO usar CQRS:**

- Operações CRUD simples
- Modelos de leitura e escrita são idênticos
- Dataset pequeno com padrões de acesso uniformes
- A equipe não está familiarizada com o padrão

### Lado de Comando (Command)

```typescript
// Command
export class PlaceOrderCommand {
  constructor(
    public readonly customerId: string,
    public readonly items: Array<{ productId: string; quantity: number }>,
  ) {}
}

// Handler
@CommandHandler(PlaceOrderCommand)
export class PlaceOrderHandler implements ICommandHandler<PlaceOrderCommand> {
  constructor(
    @Inject(ORDER_REPOSITORY) private readonly repo: OrderRepository,
    private readonly eventBus: EventBus,
  ) {}

  async execute(command: PlaceOrderCommand): Promise<string> {
    const order = new OrderAggregate(generateId(), command.customerId)
    for (const item of command.items) {
      const price = await this.pricingService.getPrice(item.productId)
      order.addItem(item.productId, item.quantity, price)
    }
    order.confirm()
    await this.repo.save(order)
    this.eventBus.publish(new OrderPlacedEvent(order.id, order.total))
    return order.id
  }
}
```

### Lado de Consulta (Query)

```typescript
// Query
export class GetOrderSummaryQuery {
  constructor(public readonly orderId: string) {}
}

// Handler — pode ignorar o modelo de domínio por performance de leitura
@QueryHandler(GetOrderSummaryQuery)
export class GetOrderSummaryHandler implements IQueryHandler<GetOrderSummaryQuery> {
  constructor(private readonly prisma: PrismaService) {}

  async execute(query: GetOrderSummaryQuery) {
    return this.prisma.order.findUnique({
      where: { id: query.orderId },
      include: { items: true },
    })
  }
}
```

---

## 4. Padrão de Serviço Simples — Default

Esta é a abordagem default recomendada. Use serviços `@Injectable()` que encapsulam a lógica de negócio e interagem com os repositórios via DI.

```typescript
// libs/orders/application/services/order.service.ts
@Injectable()
export class OrderService {
  constructor(
    @Inject(ORDER_REPOSITORY) private readonly repo: OrderRepository,
    @Inject(EVENT_PUBLISHER) private readonly events: EventPublisher,
  ) {}

  async placeOrder(customerId: string, items: Array<{ productId: string; quantity: number }>): Promise<string> {
    const order = new OrderAggregate(generateId(), customerId)
    for (const item of items) {
      const price = await this.pricingService.getPrice(item.productId)
      order.addItem(item.productId, item.quantity, price)
    }
    order.confirm()
    await this.repo.save(order)
    await this.events.publish('orders.order.placed', { orderId: order.id, total: order.total })
    return order.id
  }

  async getById(id: string): Promise<OrderAggregate> {
    const order = await this.repo.findById(id)
    if (!order) throw new OrderNotFoundError(id)
    return order
  }
}
```

O controller injeta o serviço diretamente:

```typescript
@Controller('orders')
@ApiTags('orders')
export class OrderController {
  constructor(private readonly orderService: OrderService) {}

  @Post()
  async create(@Body() dto: CreateOrderDto) {
    const orderId = await this.orderService.placeOrder(dto.customerId, dto.items)
    return { id: orderId }
  }

  @Get(':id')
  async findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.orderService.getById(id)
  }
}
```

**Quando evoluir para CQRS:** Se você notar que seu serviço está crescendo muito, com lógica de leitura e escrita bem diferentes, ou precisa otimizar leituras de forma independente (ex.: views desnormalizadas, camadas de cache), considere extrair o serviço para command handlers e query handlers separados.

---

## 5. Configuração do Workspace NX

### nx.json

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "defaultBase": "main",
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "sharedGlobals": ["{workspaceRoot}/tsconfig.base.json"],
    "production": ["default", "!{projectRoot}/**/*.spec.ts", "!{projectRoot}/jest.config.ts"]
  },
  "targetDefaults": {
    "build": { "dependsOn": ["^build"], "inputs": ["production", "^production"] },
    "test": { "inputs": ["default", "^production"] },
    "lint": { "inputs": ["default"] }
  }
}
```

### Fronteiras de Módulos (tags eslint ou project.json)

Use tags NX para reforçar as fronteiras dos módulos:

```json
// project.json para libs/billing
{ "tags": ["scope:billing", "type:domain-lib"] }

// project.json para libs/identity
{ "tags": ["scope:identity", "type:domain-lib"] }

// project.json para libs/shared
{ "tags": ["scope:shared", "type:shared-lib"] }
```

Regras de fronteira:

- `scope:billing` só pode importar de `scope:shared`
- `scope:identity` só pode importar de `scope:shared`
- Nenhum import direto entre escopos de domínio

---

## 6. Configuração Strict do TypeScript

### tsconfig.base.json

```json
{
  "compileOnSave": false,
  "compilerOptions": {
    "rootDir": ".",
    "sourceMap": true,
    "declaration": false,
    "moduleResolution": "node",
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "importHelpers": true,
    "target": "es2022",
    "module": "esnext",
    "lib": ["es2022"],
    "skipLibCheck": true,
    "skipDefaultLibCheck": true,
    "baseUrl": ".",
    "strict": true,
    "noImplicitReturns": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noUncheckedIndexedAccess": true,
    "paths": {
      "@project/shared/domain": ["libs/shared/domain/src/index.ts"],
      "@project/shared/contracts": ["libs/shared/contracts/src/index.ts"],
      "@project/shared/infrastructure": ["libs/shared/infrastructure/src/index.ts"],
      "@project/billing": ["libs/billing/src/index.ts"],
      "@project/identity": ["libs/identity/src/index.ts"],
      "@project/orders": ["libs/orders/src/index.ts"]
    }
  },
  "exclude": ["node_modules", "tmp", "dist"]
}
```

**Flags inegociáveis:** `strict`, `strictNullChecks`, `noImplicitAny`, `noUncheckedIndexedAccess`. Elas capturam bugs reais e reforçam a corretude do domínio.
