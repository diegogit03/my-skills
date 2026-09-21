# Comunicação entre Módulos

## Sumário

1. Interface de Eventos de Domínio (linha ~10)
2. Publisher In-Memory (Desenvolvimento) (linha ~40)
3. Publishers de Produção (linha ~70)
4. Contratos entre Módulos (linha ~160)
5. Padrão de Event Handler (linha ~200)
6. Escolhendo um Sistema de Eventos (linha ~240)

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

Para desenvolvimento local e testes. Troque para publishers de produção via DI.

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

## 3. Publishers de Produção

### Kafka — Para alto throughput e event sourcing

```typescript
// libs/shared/infrastructure/src/events/kafka-event-publisher.ts
import { Injectable, Logger } from '@nestjs/common'
import { Producer } from 'kafkajs'
import { EventPublisher } from '@project/shared/contracts'

@Injectable()
export class KafkaEventPublisher implements EventPublisher {
  private readonly logger = new Logger(KafkaEventPublisher.name)

  constructor(private readonly producer: Producer) {}

  async publish<T extends Record<string, unknown>>(eventName: string, payload: T): Promise<void> {
    const topic = `events.${eventName.replace(/\./g, '-')}`
    await this.producer.send({
      topic,
      messages: [
        {
          key: eventName,
          value: JSON.stringify(payload),
          headers: { eventType: eventName, timestamp: new Date().toISOString() },
        },
      ],
    })
    this.logger.log(`Evento publicado em ${topic}: ${eventName}`)
  }
}
```

### SQS — Para fila simples, nativa AWS

```typescript
// libs/shared/infrastructure/src/events/sqs-event-publisher.ts
import { Injectable } from '@nestjs/common'
import { SQSClient, SendMessageCommand } from '@aws-sdk/client-sqs'
import { EventPublisher } from '@project/shared/contracts'

@Injectable()
export class SQSEventPublisher implements EventPublisher {
  constructor(
    private readonly sqsClient: SQSClient,
    private readonly queueUrlResolver: QueueUrlResolver,
  ) {}

  async publish<T extends Record<string, unknown>>(eventName: string, payload: T): Promise<void> {
    const queueUrl = this.queueUrlResolver.resolve(eventName)
    await this.sqsClient.send(
      new SendMessageCommand({
        QueueUrl: queueUrl,
        MessageBody: JSON.stringify(payload),
        MessageAttributes: {
          eventType: { StringValue: eventName, DataType: 'String' },
        },
      }),
    )
  }
}
```

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

### Registrando Publishers via DI

```typescript
// libs/shared/infrastructure/src/events/event-publisher.module.ts
@Module({
  providers: [
    {
      provide: EVENT_PUBLISHER,
      useFactory: (config: ConfigService) => {
        const driver = config.get('EVENT_DRIVER', 'in-memory')
        switch (driver) {
          case 'kafka':
            return new KafkaEventPublisher(/* ... */)
          case 'sqs':
            return new SQSEventPublisher(/* ... */)
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

## 4. Contratos entre Módulos

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

## 5. Padrão de Event Handler

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

## 6. Escolhendo um Sistema de Eventos

| Critério        | Kafka                      | SQS                      | Redis              | In-Memory     |
| --------------- | -------------------------- | ------------------------ | ------------------ | ------------- |
| **Entrega**     | At-least-once              | At-least-once            | At-most-once       | Best-effort   |
| **Ordenação**   | Por partição               | Filas FIFO               | Sem garantia       | Síncrona      |
| **Persistência**| Sim (configurável)         | Sim (14 dias)            | Não                | Não           |
| **Throughput**  | Muito alto                 | Alto                     | Muito alto         | N/A           |
| **Complexidade**| Alta                       | Baixa                    | Baixa              | Mínima        |
| **Caso de Uso** | Event sourcing, alta escala| Nativo AWS, fluxos simples| Tempo real, pub/sub | Apenas dev/test |

**Recomendação:** Comece com In-Memory para desenvolvimento, SQS ou Redis para produção (dependendo do provedor de nuvem), Kafka apenas quando superar soluções mais simples.
