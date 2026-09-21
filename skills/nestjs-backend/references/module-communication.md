# Comunicação entre Módulos

## Sumário

1. Public API — Comunicação Síncrona (linha ~9)
2. Eventos — Comunicação Assíncrona (linha ~90)

---

## 1. Public API — Comunicação Síncrona

Quando um módulo precisa de uma resposta **na mesma requisição** (consulta ou comando com retorno), use o padrão Public API: a interface fica em `@common/contracts`, o módulo dono implementa e o módulo consumidor depende apenas da interface — nunca da classe concreta.

```typescript
// src/common/contracts/api/identity.api.ts
export interface IdentityPublicApi {
  getUser(userId: string): Promise<{ id: string; name: string; email: string } | null>
  userExists(userId: string): Promise<boolean>
}

export const IDENTITY_PUBLIC_API = Symbol('IdentityPublicApi')
```

O módulo dono implementa a interface e registra no token:

```typescript
// src/modules/identity/identity.api.ts
import { Injectable } from '@nestjs/common'
import { IdentityPublicApi, IDENTITY_PUBLIC_API } from '@common/contracts/api/identity.api'
import { SessionService } from './session/session.service'

@Injectable()
export class IdentityApi implements IdentityPublicApi {
  constructor(private readonly sessionService: SessionService) {}

  async getUser(userId: string) {
    const user = await this.sessionService.findById(userId)
    if (!user) return null
    return { id: user.id, name: user.name, email: user.email } // DTO plano, nunca a entidade
  }

  async userExists(userId: string) {
    return (await this.sessionService.findById(userId)) !== null
  }
}

// src/modules/identity/identity.module.ts
@Module({
  providers: [
    SessionService,
    { provide: IDENTITY_PUBLIC_API, useExisting: IdentityApi },
  ],
  exports: [IDENTITY_PUBLIC_API],
})
export class IdentityModule {}
```

O consumidor injeta apenas o token:

```typescript
// src/modules/finance/wallets/wallet.service.ts
import { Inject } from '@nestjs/common'
import { IDENTITY_PUBLIC_API, IdentityPublicApi } from '@common/contracts/api/identity.api'

@Injectable()
export class WalletService {
  constructor(
    @Inject(IDENTITY_PUBLIC_API) private readonly identity: IdentityPublicApi,
  ) {}

  async create(userId: string, name: string) {
    if (!(await this.identity.userExists(userId))) {
      throw new UserNotFoundError(userId)
    }
    // ...
  }
}
```

**Regras da Public API:**

- A interface e o token vivem **sempre** em `@common/contracts/api/` — o consumidor não conhece o módulo provedor
- Métodos retornam **DTOs planos** (primitivos/serializáveis) — nunca entidades TypeORM nem classes de domínio
- Interface enxuta: só o que outros módulos realmente usam (é uma facade do módulo)
- Síncrono dentro do agregado/módulo; entre módulos, use Public API apenas quando a resposta é necessária na mesma requisição
- Erros de negócio do provedor são convertidos no adapter (`IdentityApi`) para erros de domínio do consumidor ou `null`/`Result`

---

## 2. Eventos — Comunicação Assíncrona

Para notificações entre módulos que **não precisam de resposta na mesma requisição** (ex.: "usuário criado" → configurar finance). Eventos aqui são simples notificações fire-and-forget — não são necessariamente eventos de domínio: podem indicar fluxos internos, integrações ou efeitos colaterais.

Interface única no `common`:

```typescript
// src/common/contracts/events/event-publisher.interface.ts
export interface EventPublisher {
  publish(eventName: string, payload: Record<string, unknown>): Promise<void>
}

export const EVENT_PUBLISHER = Symbol('EventPublisher')
```

Publisher in-memory (dev) e registro via DI:

```typescript
// src/common/infrastructure/events/event-publisher.module.ts
import { Injectable, Module } from '@nestjs/common'
import { EventEmitter2, EventEmitterModule } from '@nestjs/event-emitter'
import { EventPublisher, EVENT_PUBLISHER } from '@common/contracts'

@Injectable()
export class InMemoryEventPublisher implements EventPublisher {
  constructor(private readonly emitter: EventEmitter2) {}

  async publish(eventName: string, payload: Record<string, unknown>): Promise<void> {
    this.emitter.emit(eventName, payload)
  }
}

@Module({
  imports: [EventEmitterModule],
  providers: [
    { provide: EVENT_PUBLISHER, useClass: InMemoryEventPublisher },
  ],
  exports: [EVENT_PUBLISHER],
})
export class EventPublisherModule {}
```

O módulo publica onde faz sentido (service, handler):

```typescript
// src/modules/finance/wallets/wallet.service.ts
import { Inject } from '@nestjs/common'
import { EVENT_PUBLISHER, EventPublisher } from '@common/contracts'

@Injectable()
export class WalletService {
  constructor(
    @Inject(EVENT_PUBLISHER) private readonly events: EventPublisher,
  ) {}

  async create(userId: string, name: string) {
    const wallet = await this.walletRepository.save(new Wallet(userId, name))
    await this.events.publish('finance.wallet.created', { walletId: wallet.id, userId })
    return wallet
  }
}
```

Outros módulos reagem com handlers `@OnEvent` — sempre idempotentes:

```typescript
// src/modules/notifications/handlers/on-wallet-created.handler.ts
import { Injectable, Logger } from '@nestjs/common'
import { OnEvent } from '@nestjs/event-emitter'

@Injectable()
export class OnWalletCreatedHandler {
  private readonly logger = new Logger(OnWalletCreatedHandler.name)

  @OnEvent('finance.wallet.created')
  async handle(payload: { walletId: string; userId: string }): Promise<void> {
    this.logger.log(`Wallet criada: ${payload.walletId}`)
    // idempotência: verifique se a ação já foi executada antes de executar
  }
}
```

**Regras dos eventos:**

- Nome em notação de pontos: `module.aggregate.action`
- Payload apenas com dados serializáveis (primitivos, IDs) — nunca entidades de domínio
- Handler reage ao payload, sempre idempotente (use IDs como chave de deduplicação)
- O publisher in-memory não sobrevive a restarts nem escala entre instâncias — em produção, troque a implementação (Redis, fila) via DI sem mudar o contrato
