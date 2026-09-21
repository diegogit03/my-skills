# Padrões de Teste

## Sumário

1. Testes da Camada de Domínio (linha ~10)
2. Testes de Serviço / Handler (linha ~60)
3. Testes de Controller (linha ~130)
4. Testes de Integração de Módulo (linha ~180)
5. Testes E2E (linha ~220)
6. Mock Factories (linha ~280)

---

## 1. Testes da Camada de Domínio

Entidades de domínio devem ser testadas de forma pura — sem dependências de framework.

```typescript
// libs/billing/domain/__tests__/billing-plan.entity.spec.ts
import { BillingPlan, BillingInterval } from '../entities/billing-plan.entity'

describe('BillingPlan', () => {
  const createPlan = (overrides?: Partial<ConstructorParameters<typeof BillingPlan>>) =>
    new BillingPlan(
      'plan-1',
      'Pro Plan',
      2999,
      BillingInterval.MONTHLY,
      new Date('2024-01-01'),
      ...Object.values(overrides ?? {}),
    )

  describe('isActive', () => {
    it('retorna true quando o preço é positivo', () => {
      const plan = createPlan()
      expect(plan.isActive()).toBe(true)
    })

    it('retorna false quando o preço é zero', () => {
      const plan = new BillingPlan('p-1', 'Free', 0, BillingInterval.MONTHLY, new Date())
      expect(plan.isActive()).toBe(false)
    })
  })

  describe('canUpgradeTo', () => {
    it('permite upgrade para um plano mais caro', () => {
      const basic = new BillingPlan('p-1', 'Basic', 999, BillingInterval.MONTHLY, new Date())
      const pro = new BillingPlan('p-2', 'Pro', 2999, BillingInterval.MONTHLY, new Date())
      expect(basic.canUpgradeTo(pro)).toBe(true)
    })

    it('rejeita downgrade', () => {
      const pro = new BillingPlan('p-2', 'Pro', 2999, BillingInterval.MONTHLY, new Date())
      const basic = new BillingPlan('p-1', 'Basic', 999, BillingInterval.MONTHLY, new Date())
      expect(pro.canUpgradeTo(basic)).toBe(false)
    })
  })
})
```

---

## 2. Testes de Serviço / Handler

Teste a lógica de negócio por meio de serviços (default) ou handlers (CQRS). Mocke as interfaces de repositório, não o Prisma.

### Teste de Serviço (Abordagem Default)

```typescript
// libs/billing/application/__tests__/billing-plan.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing'
import { BillingPlanService } from '../services/billing-plan.service'
import { BILLING_PLAN_REPOSITORY } from '../../domain/repositories/billing-plan.repository'
import { EVENT_PUBLISHER } from '@project/shared/contracts'

describe('BillingPlanService', () => {
  let service: BillingPlanService
  let repository: jest.Mocked<BillingPlanRepository>
  let events: jest.Mocked<EventPublisher>

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        BillingPlanService,
        {
          provide: BILLING_PLAN_REPOSITORY,
          useValue: { findById: jest.fn(), findAll: jest.fn(), save: jest.fn(), delete: jest.fn() },
        },
        {
          provide: EVENT_PUBLISHER,
          useValue: { publish: jest.fn() },
        },
      ],
    }).compile()

    service = module.get(BillingPlanService)
    repository = module.get(BILLING_PLAN_REPOSITORY)
    events = module.get(EVENT_PUBLISHER)
  })

  afterEach(() => jest.clearAllMocks())

  it('cria um plano de cobrança e publica evento', async () => {
    repository.save.mockImplementation(async (plan) => plan)

    const result = await service.create('Pro', 2999, BillingInterval.MONTHLY)

    expect(result.name).toBe('Pro')
    expect(result.priceInCents).toBe(2999)
    expect(repository.save).toHaveBeenCalledTimes(1)
    expect(events.publish).toHaveBeenCalledWith(
      'billing.plan.created',
      expect.objectContaining({ planId: expect.any(String) }),
    )
  })

  it('lança erro quando o plano não é encontrado', async () => {
    repository.findById.mockResolvedValue(null)
    await expect(service.findById('nonexistent')).rejects.toThrow(BillingPlanNotFoundError)
  })
})
```

### Teste de Handler CQRS (Quando Usar CQRS)

```typescript
// libs/billing/application/__tests__/create-billing-plan.handler.spec.ts
import { Test, TestingModule } from '@nestjs/testing'
import { EventBus } from '@nestjs/cqrs'
import { CreateBillingPlanHandler } from '../handlers/create-billing-plan.handler'
import { CreateBillingPlanCommand } from '../commands/create-billing-plan.command'
import { BILLING_PLAN_REPOSITORY } from '../../domain/repositories/billing-plan.repository'

describe('CreateBillingPlanHandler', () => {
  let handler: CreateBillingPlanHandler
  let repository: jest.Mocked<BillingPlanRepository>
  let eventBus: jest.Mocked<EventBus>

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        CreateBillingPlanHandler,
        { provide: BILLING_PLAN_REPOSITORY, useValue: { findById: jest.fn(), save: jest.fn(), delete: jest.fn() } },
        { provide: EventBus, useValue: { publish: jest.fn() } },
      ],
    }).compile()

    handler = module.get(CreateBillingPlanHandler)
    repository = module.get(BILLING_PLAN_REPOSITORY)
    eventBus = module.get(EventBus)
  })

  afterEach(() => jest.clearAllMocks())

  it('cria um plano de cobrança e publica evento', async () => {
    const command = new CreateBillingPlanCommand('Pro', 2999, BillingInterval.MONTHLY)
    repository.save.mockImplementation(async (plan) => plan)

    const result = await handler.execute(command)

    expect(result.name).toBe('Pro')
    expect(repository.save).toHaveBeenCalledTimes(1)
    expect(eventBus.publish).toHaveBeenCalledTimes(1)
  })
})
```

---

## 3. Testes de Controller

Teste a camada HTTP de forma independente dos serviços.

```typescript
// libs/billing/presentation/__tests__/billing-plan.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing'
import { BillingPlanController } from '../billing-plan.controller'
import { BillingPlanService } from '../../application/services/billing-plan.service'

describe('BillingPlanController', () => {
  let controller: BillingPlanController
  let service: jest.Mocked<BillingPlanService>

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [BillingPlanController],
      providers: [
        { provide: BillingPlanService, useValue: { create: jest.fn(), findById: jest.fn(), findAll: jest.fn() } },
      ],
    }).compile()

    controller = module.get(BillingPlanController)
    service = module.get(BillingPlanService)
  })

  it('cria um plano via serviço', async () => {
    const expectedPlan = { id: 'p-1', name: 'Pro', priceInCents: 2999 }
    service.create.mockResolvedValue(expectedPlan as any)

    const dto = { name: 'Pro', priceInCents: 2999, interval: 'MONTHLY' }
    const result = await controller.create(dto as any)

    expect(service.create).toHaveBeenCalledTimes(1)
    expect(result).toBeDefined()
  })
})
```

---

## 4. Testes de Integração de Módulo

Teste que os módulos funcionam corretamente DENTRO de suas fronteiras. Estes testes verificam que DI, repositórios e serviços funcionam juntos.

```typescript
// libs/billing/__tests__/billing.module.integration.spec.ts
import { Test, TestingModule } from '@nestjs/testing'
import { BillingModule } from '../billing.module'
import { PrismaModule } from '@project/shared/infrastructure'
import { BillingPlanService } from '../application/services/billing-plan.service'
import { BILLING_PLAN_REPOSITORY } from '../domain/repositories/billing-plan.repository'

describe('BillingModule (integração)', () => {
  let module: TestingModule

  beforeAll(async () => {
    module = await Test.createTestingModule({
      imports: [BillingModule, PrismaModule],
    }).compile()
  })

  afterAll(async () => {
    await module.close()
  })

  it('resolve BillingPlanService', () => {
    const service = module.get(BillingPlanService)
    expect(service).toBeDefined()
  })

  it('resolve BillingPlanRepository', () => {
    const repo = module.get(BILLING_PLAN_REPOSITORY)
    expect(repo).toBeDefined()
  })
})
```

---

## 5. Testes E2E

Teste o ciclo de vida HTTP completo, incluindo auth, validação e resposta.

```typescript
// apps/api/test/billing.e2e-spec.ts
import { INestApplication } from '@nestjs/common'
import { Test } from '@nestjs/testing'
import * as request from 'supertest'
import { AppModule } from '../src/app.module'
import { ValidationPipe } from '@nestjs/common'

describe('Billing (e2e)', () => {
  let app: INestApplication
  let authToken: string

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],
    }).compile()

    app = module.createNestApplication()
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }))
    await app.init()

    // Obter token de autenticação
    const authResponse = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'admin@test.com', password: 'TestPass123!' })
    authToken = authResponse.body.access_token
  })

  afterAll(() => app.close())

  it('POST /billing/plans — cria plano', async () => {
    const response = await request(app.getHttpServer())
      .post('/billing/plans')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ name: 'Pro', priceInCents: 2999, interval: 'MONTHLY' })
      .expect(201)

    expect(response.body).toHaveProperty('id')
    expect(response.body.name).toBe('Pro')
  })

  it('POST /billing/plans — rejeita dados inválidos', async () => {
    await request(app.getHttpServer())
      .post('/billing/plans')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ name: '', priceInCents: -1 })
      .expect(400)
  })

  it('GET /billing/plans — retorna 401 sem autenticação', async () => {
    await request(app.getHttpServer()).get('/billing/plans').expect(401)
  })
})
```

---

## 6. Mock Factories

Criadores de mocks reutilizáveis para configuração consistente de testes.

```typescript
// libs/shared/infrastructure/src/testing/mock-factories.ts

export function createMockRepository<T>() {
  return {
    findById: jest.fn(),
    findAll: jest.fn(),
    save: jest.fn(),
    delete: jest.fn(),
  } as jest.Mocked<any>
}

export function createMockEventPublisher() {
  return { publish: jest.fn() } as jest.Mocked<any>
}

export function createMockService(methods: string[]) {
  return Object.fromEntries(methods.map((m) => [m, jest.fn()])) as jest.Mocked<any>
}
```

## Referência Rápida

| Nível de Teste | O Que Testar                      | Onde                                    | Dependências             |
| -------------- | --------------------------------- | --------------------------------------- | ------------------------ |
| Domínio        | Lógica de entidades, value objects| `libs/[module]/domain/__tests__/`       | Nenhuma                  |
| Serviço        | Regras de negócio via serviços    | `libs/[module]/application/__tests__/`  | Repos + eventos mockados |
| Handler        | Regras de negócio via handlers CQRS| `libs/[module]/application/__tests__/` | Repos + event bus mockados|
| Controller     | Interface HTTP                    | `libs/[module]/presentation/__tests__/` | Serviço mockado          |
| Integração     | DI do módulo, cadeia completa     | `libs/[module]/__tests__/`              | Módulo real, banco de teste |
| E2E            | Ciclo de vida HTTP completo       | `apps/api/test/`                        | App completo, banco de teste |
