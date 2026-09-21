# Padrões de Arquitetura

## Sumário

1. Padrão de Serviço Simples — Default (linha ~10)
2. Configuração Strict do TypeScript (linha ~90)

---

## 1. Padrão de Serviço Simples — Default

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

---

## 2. Configuração Strict do TypeScript

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
