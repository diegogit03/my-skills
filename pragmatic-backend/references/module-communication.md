# Comunicação entre Módulos

## Sumário

1. Interface de Eventos de Domínio (linha ~10)
2. Publisher In-Memory (Desenvolvimento) (linha ~40)
3. Publisher de Produção — Redis (linha ~70)
4. Registrando Publishers via DI (linha ~100)
5. Contratos entre Módulos (linha ~125)
6. Padrão de Event Handler (linha ~165)
7. Escolhendo um Sistema de Eventos (linha ~200)

---

## 1. Interface de Eventos de Domínio

Todos os eventos implementam uma interface compartilhada. Este é o contrato entre módulos.

```typescript
// libs/shared/contracts/src/events/domain-event.interface.ts
export interface DomainEvent {
  readonly aggregateId: string
  readonly eventType: string
  readonly occurredAt: Date
  readonly version: number
  readonly payload: Record<string, unknown>
}

// libs/shared/contracts/src/events/event-publisher.interface.ts
export interface EventPublisher {
  publish<T extends Record<string, unknown>>(eventName: string, payload: T): Promise<void>
}

export const EVENT_PUBLISHER = Symbol('EventPublisher')
```

## 2. Publisher In-Memory (Desenvolvimento)

Para desenvolvimento local e testes. Troque para o publisher de produção via DI.

```typescript
// libs/shared/infrastructure/src/events/in-memory-event-publisher.ts
import { Injectable, Logger } from '@nestjs/common'
import { EventEmitter2 } from '@nestjs/event-emitter'
import { EventPublisher } from '@project/shared/contracts'

@Injectable()
export class InMemoryEventPublisher implements EventPublisher {
  private readonly logger = new Logger(InMemoryEventPublisher.name)

  constructor(private readonly eventEmitter: EventEmitter2) {}

  async publish<T extends Record<string, unknown>>(eventName: string, payload: T): Promise<void> {
    this.logger.debug(`Publicando evento: ${eventName}`, payload)
    this.eventEmitter.emit(eventName, payload)
  }
}
```

> ⚠️ **Nunca use eventos in-memory para comunicação entre módulos em produção.** Eventos in-memory não sobrevivem a restarts do processo, não escalam entre instâncias e não têm garantias de entrega.

---

## 3. Publisher de Produção — Redis

### Redis — Para cenários de tempo real, pub/sub

```typescript
// libs/shared/infrastructure/src/events/redis-event-publisher.ts
import { Injectable } from '@nestjs/common'
import { Redis } from 'ioredis'
import { EventPublisher } from '@project/shared/contracts'

@Injectable()
export class RedisEventPublisher implements EventPublisher {
  constructor(private readonly redis: Redis) {}

  async publish<T extends Record<string, unknown>>(eventName: string, payload: T): Promise<void> {
    await this.redis.publish(eventName, JSON.stringify(payload))
  }
}
```

---

## 4. Registrando Publishers via DI

```typescript
// libs/shared/infrastructure/src/events/event-publisher.module.ts
@Module({
  providers: [
    {
      provide: EVENT_PUBLISHER,
      useFactory: (config: ConfigService) => {
        const driver = config.get('EVENT_DRIVER', 'in-memory')
        switch (driver) {
          case 'redis':
            return new RedisEventPublisher(/* ... */)
          default:
            return new InMemoryEventPublisher(/* ... */)
        }
      },
      inject: [ConfigService],
    },
  ],
  exports: [EVENT_PUBLISHER],
})
export class EventPublisherModule {}
```

---

## 5. Contratos entre Módulos

Eventos são a ÚNICA forma pela qual os módulos devem se comunicar. Defina os contratos de eventos na biblioteca de contratos compartilhados.

```typescript
// libs/shared/contracts/src/events/identity.events.ts
export class IdentityUserCreatedEvent implements DomainEvent {
  readonly eventType = 'identity.user.created'
  readonly version = 1
  readonly occurredAt = new Date()

  constructor(
    public readonly aggregateId: string,
    public readonly payload: { userId: string; email: string; name: string },
  ) {}
}

// libs/shared/contracts/src/events/billing.events.ts
export class BillingSubscriptionActivatedEvent implements DomainEvent {
  readonly eventType = 'billing.subscription.activated'
  readonly version = 1
  readonly occurredAt = new Date()

  constructor(
    public readonly aggregateId: string,
    public readonly payload: { subscriptionId: string; planId: string; userId: string },
  ) {}
}
```

**Regras para contratos de eventos:**

- Tipos de evento usam notação de pontos: `module.aggregate.action`
- Payloads contêm apenas dados serializáveis e primitivos
- Nunca inclua referências a entidades de domínio em eventos (use IDs)
- Versione eventos quando seu schema mudar
- Eventos são imutáveis após a criação

---

## 6. Padrão de Event Handler

Handlers em outros módulos reagem a eventos. Sempre idempotentes.

```typescript
// libs/billing/application/handlers/on-user-created.handler.ts
import { OnEvent } from '@nestjs/event-emitter'
import { Injectable, Logger } from '@nestjs/common'

@Injectable()
export class OnUserCreatedHandler {
  private readonly logger = new Logger(OnUserCreatedHandler.name)

  @OnEvent('identity.user.created')
  async handle(event: IdentityUserCreatedEvent): Promise<void> {
    this.logger.log(`Configurando billing para o usuário: ${event.payload.userId}`)
    await this.billingService.createDefaultProfile(event.payload.userId)
  }
}
```

**Regras de idempotência:**

- Verifique se a ação já foi executada antes de executar
- Use `aggregateId` + `eventType` como chave de deduplicação
- Registre todo o processamento de eventos para observabilidade

---

## 7. Escolhendo um Sistema de Eventos

| Critério        | Redis              | In-Memory     |
| --------------- | ------------------ | ------------- |
| **Entrega**     | At-most-once       | Best-effort   |
| **Ordenação**   | Sem garantia       | Síncrona      |
| **Persistência**| Não                | Não           |
| **Throughput**  | Muito alto         | N/A           |
| **Complexidade**| Baixa              | Mínima        |
| **Caso de Uso** | Tempo real, pub/sub| Apenas dev/test |

**Recomendação:** Comece com In-Memory para desenvolvimento e Redis para produção.
