# Padrões de Teste

> Usa **Vitest** como framework de testes. Execução: `vitest` (watch), `vitest run` (CI). API compatível com Jest (`describe`, `it`, `expect`, `vi.fn()`), mas com execução mais rápida e suporte nativo a ESM/TS.

## Sumário

1. Testes de Entidade (linha ~14)
2. Testes de Serviço (linha ~55)
3. Testes E2E (linha ~117)
4. Mock Factories (linha ~175)

---

## 1. Testes de Entidade

Quando a entidade **não é anêmica** (contém comportamento de negócio, invariants e factories), teste-a isoladamente como objeto puro — sem Nest, sem mocks.

```typescript
// src/modules/finance/core/wallet.entity.spec.ts
import { describe, it, expect } from 'vitest'
import { Wallet } from './wallet.entity'

describe('Wallet', () => {
  it('cria wallet com saldo inicial zero', () => {
    const wallet = new Wallet('user-1', 'Main')

    expect(wallet.name).toBe('Main')
    expect(wallet.balance).toBe(0)
    expect(wallet.id).toBeDefined()
  })

  it('não permite criar wallet com nome vazio', () => {
    expect(() => new Wallet('user-1', '')).toThrow()
  })

  it('deposita e registra evento de domínio', () => {
    const wallet = new Wallet('user-1', 'Main')

    wallet.deposit(100)

    expect(wallet.balance).toBe(100)
    expect(wallet.pullDomainEvents()).toContainEqual(
      expect.objectContaining({ type: 'wallet.deposited' }),
    )
  })

  it('não permite saque acima do saldo', () => {
    const wallet = new Wallet('user-1', 'Main')

    expect(() => wallet.withdraw(50)).toThrow(InsufficientBalanceError)
  })
})
```

## 2. Testes de Serviço

Teste a lógica de negócio por meio de serviços. Mocke os repositórios (classes concretas) via `useValue`, não o TypeORM. O spec fica no mesmo nível do service.

```typescript
// src/modules/finance/core/service/wallet.service.spec.ts
import { describe, it, beforeEach, afterEach, vi, expect } from 'vitest'
import { Test } from '@nestjs/testing'
import type { TestingModule } from '@nestjs/testing'
import { WalletService } from './wallet.service'
import { WalletRepository } from '../../../persistence/repository/wallet.repository'
import { EVENT_PUBLISHER } from '@common/contracts'

describe('WalletService', () => {
  let service: WalletService
  let walletRepository: Record<'findById' | 'findAll' | 'save' | 'delete', ReturnType<typeof vi.fn>>
  let events: { publish: ReturnType<typeof vi.fn> }

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        WalletService,
        {
          provide: WalletRepository,
          useValue: { findById: vi.fn(), findAll: vi.fn(), save: vi.fn(), delete: vi.fn() },
        },
        {
          provide: EVENT_PUBLISHER,
          useValue: { publish: vi.fn() },
        },
      ],
    }).compile()

    service = module.get(WalletService)
    walletRepository = module.get(WalletRepository)
    events = module.get(EVENT_PUBLISHER)
  })

  afterEach(() => vi.clearAllMocks())

  it('cria uma wallet e publica evento', async () => {
    walletRepository.save.mockImplementation(async (wallet) => wallet)

    const result = await service.create('user-1', 'Main')

    expect(result.name).toBe('Main')
    expect(walletRepository.save).toHaveBeenCalledTimes(1)
    expect(events.publish).toHaveBeenCalledWith(
      'finance.wallet.created',
      expect.objectContaining({ walletId: expect.any(String) }),
    )
  })

  it('lança erro quando a wallet não é encontrada', async () => {
    walletRepository.findById.mockResolvedValue(null)
    await expect(service.getById('nonexistent')).rejects.toThrow(WalletNotFoundError)
  })
})
```

---

## 3. Testes E2E (por módulo)

Teste o ciclo de vida HTTP completo, incluindo auth, validação e resposta. Ficam por módulo, em `src/modules/[m]/__tests__/`.

```typescript
// src/modules/finance/__tests__/finance.e2e-spec.ts
import { describe, it, beforeAll, afterAll, expect } from 'vitest'
import { INestApplication, ValidationPipe } from '@nestjs/common'
import { Test } from '@nestjs/testing'
import request from 'supertest'
import { AppModule } from '../../../app.module'

describe('Finance (e2e)', () => {
  let app: INestApplication
  let authToken: string

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],
    }).compile()

    app = module.createNestApplication()
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }))
    await app.init()

    // Obter token de autenticação (Personal Access Token)
    authToken = await seedTestToken()
  })

  afterAll(() => app.close())

  it('POST /wallets — cria wallet', async () => {
    const response = await request(app.getHttpServer())
      .post('/wallets')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ name: 'Main' })
      .expect(201)

    expect(response.body).toHaveProperty('id')
    expect(response.body.name).toBe('Main')
  })

  it('POST /wallets — rejeita dados inválidos', async () => {
    await request(app.getHttpServer())
      .post('/wallets')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ name: '' })
      .expect(400)
  })

  it('GET /wallets — retorna 401 sem autenticação', async () => {
    await request(app.getHttpServer()).get('/wallets').expect(401)
  })
})
```

---

## 4. Mock Factories

Criadores de mocks reutilizáveis para configuração consistente de testes.

```typescript
// src/common/infrastructure/testing/mock-factories.ts
import { vi } from 'vitest'

export function createMockRepository() {
  return {
    findById: vi.fn(),
    findAll: vi.fn(),
    save: vi.fn(),
    delete: vi.fn(),
  }
}

export function createMockEventPublisher() {
  return { publish: vi.fn() }
}

export function createMockService(methods: string[]) {
  return Object.fromEntries(methods.map((m) => [m, vi.fn()]))
}
```

## Referência Rápida

| Nível de Teste | O Que Testar                      | Onde                                             | Dependências             |
| -------------- | --------------------------------- | ------------------------------------------------ | ------------------------ |
| Entidade       | Comportamento de domínio (rich)   | `src/modules/[m]/core/domain/*.spec.ts`          | Nenhuma (objeto puro)    |
| Serviço        | Regras de negócio via serviços    | `src/modules/[m]/core/service/*.spec.ts`         | Repos + eventos mockados |
| E2E            | Ciclo de vida HTTP completo       | `src/modules/[m]/__tests__/`                     | App completo, banco de teste |

## Configuração

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    include: ['src/**/*.spec.ts'],
    projects: [
      { test: { name: 'unit', include: ['src/**/*.spec.ts'], exclude: ['src/**/__tests__/**'] } },
      { test: { name: 'e2e', include: ['src/**/__tests__/**/*.e2e-spec.ts'], testTimeout: 30_000 } },
    ],
  },
})
```

## Comandos

```bash
vitest run                    # roda todos os testes (CI)
vitest                        # watch mode
vitest run --project unit     # apenas unitários
vitest run --project e2e      # apenas e2e
vitest run finance            # filtrar por nome de arquivo
```
