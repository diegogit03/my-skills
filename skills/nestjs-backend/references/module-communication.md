# Comunicação entre Módulos

## Sumário

1. Facade — Comunicação Síncrona (linha ~9)
2. Eventos — Comunicação Síncrona In-Process (linha ~90)

---

## 1. Facade — Comunicação Síncrona

Quando um módulo precisa de uma resposta **na mesma requisição** (consulta ou comando com retorno), use uma **Facade concreta**: uma classe `@Injectable()` que expõe a API pública do módulo, exportada diretamente pelo módulo. Não há interface intermediária nem Symbol — o consumidor importa o módulo e injeta a facade.

```typescript
// src/modules/identity/identity.facade.ts
import { Injectable } from '@nestjs/common'
import { SessionService } from './session/session.service'

@Injectable()
export class IdentityFacade {
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
```

O módulo exporta a facade diretamente:

```typescript
// src/modules/identity/identity.module.ts
@Module({
  providers: [SessionService, IdentityFacade],
  exports: [IdentityFacade],
})
export class IdentityModule {}
```

O consumidor importa o módulo e injeta a facade:

```typescript
// src/modules/finance/finance.module.ts
import { IdentityModule } from '@modules/identity/identity.module'

@Module({
  imports: [IdentityModule],
  // ...
})
export class FinanceModule {}
```

```typescript
// src/modules/finance/wallets/wallet.service.ts
import { Injectable } from '@nestjs/common'
import { IdentityFacade } from '@modules/identity/identity.facade'

@Injectable()
export class WalletService {
  constructor(private readonly identity: IdentityFacade) {}

  async create(userId: string, name: string) {
    if (!(await this.identity.userExists(userId))) {
      throw new UserNotFoundError(userId)
    }
    // ...
  }
}
```

**Regras da Facade:**

- A facade é uma **classe concreta** `@Injectable()` na pasta raiz do módulo (`<modulo>.facade.ts`)
- O módulo exporta **apenas a facade** — serviços internos ficam encapsulados
- Métodos retornam **DTOs planos** (primitivos/serializáveis) — nunca entidades TypeORM nem classes de domínio
- Interface enxuta: só o que outros módulos realmente usam (é a porta de entrada do módulo)
- Síncrono dentro do agregado/módulo; entre módulos, use facade apenas quando a resposta é necessária na mesma requisição
- Erros de negócio do provedor são convertidos na facade para erros de domínio do consumidor ou `null`/`Result`

---

## 2. Eventos — Comunicação Síncrona In-Process

Para desacoplar reações dentro do **mesmo processo**: o handler executa na mesma requisição, na mesma instância (via `EventEmitter2`, execução síncrona). Não há fila nem entrega entre processos. Eventos aqui são simples notificações — não são necessariamente eventos de domínio: podem indicar fluxos internos, integrações ou efeitos colaterais.

Cada evento é uma **classe com contrato tipado** no `common`. A base garante o nome do evento:

```typescript
// src/common/contracts/events/app-event.interface.ts
export interface AppEvent {
  readonly eventName: string
}
```

```typescript
// src/common/contracts/events/finance.events.ts
import { AppEvent } from './app-event.interface'

export class WalletCreatedEvent implements AppEvent {
  static readonly EVENT_NAME = 'finance.wallet.created'
  readonly eventName = WalletCreatedEvent.EVENT_NAME

  constructor(
    readonly walletId: string,
    readonly userId: string,
  ) {}
}
```

O publisher é uma **classe concreta** no `common` — mesmo padrão dos repositórios: injeta-se o tipo, sem interface nem token Symbol. O emit é feito pelo `eventName` da classe:

```typescript
// src/common/infrastructure/events/event-publisher.ts
import { Injectable } from '@nestjs/common'
import { EventEmitter2 } from '@nestjs/event-emitter'
import { AppEvent } from '@common/contracts/events/app-event.interface'

@Injectable()
export class EventPublisher {
  constructor(private readonly emitter: EventEmitter2) {}

  async publish(event: AppEvent): Promise<void> {
    this.emitter.emit(event.eventName, event)
  }
}

// src/common/infrastructure/events/event-publisher.module.ts
import { Module } from '@nestjs/common'
import { EventEmitterModule } from '@nestjs/event-emitter'
import { EventPublisher } from './event-publisher'

@Module({
  imports: [EventEmitterModule],
  providers: [EventPublisher],
  exports: [EventPublisher],
})
export class EventPublisherModule {}
```

O módulo publica a instância do evento onde faz sentido (service, handler):

```typescript
// src/modules/finance/wallets/wallet.service.ts
import { EventPublisher } from '@common/infrastructure/events/event-publisher'
import { WalletCreatedEvent } from '@common/contracts/events/finance.events'

@Injectable()
export class WalletService {
  constructor(
    private readonly events: EventPublisher,
  ) {}

  async create(userId: string, name: string) {
    const wallet = await this.walletRepository.save(new Wallet(userId, name))
    await this.events.publish(new WalletCreatedEvent(wallet.id, userId))
    return wallet
  }
}
```

Outros módulos reagem com handlers `@OnEvent` — executam na mesma requisição, então devem ser rápidos e idempotentes. O handler recebe o evento tipado:

```typescript
// src/modules/notifications/handlers/on-wallet-created.handler.ts
import { Injectable, Logger } from '@nestjs/common'
import { OnEvent } from '@nestjs/event-emitter'
import { WalletCreatedEvent } from '@common/contracts/events/finance.events'

@Injectable()
export class OnWalletCreatedHandler {
  private readonly logger = new Logger(OnWalletCreatedHandler.name)

  @OnEvent(WalletCreatedEvent.EVENT_NAME)
  async handle(event: WalletCreatedEvent): Promise<void> {
    this.logger.log(`Wallet criada: ${event.walletId}`)
    // idempotência: verifique se a ação já foi executada antes de executar
  }
}
```

**Regras dos eventos:**

- Cada evento é uma **classe** em `@common/contracts/events/<modulo>.events.ts`, com `EVENT_NAME` estático em notação de pontos: `module.aggregate.action`
- Dados do evento são **propriedades tipadas** no construtor — apenas serializáveis (primitivos, IDs), nunca entidades de domínio
- Publisher e handler trocam a **instância da classe** — nunca `payload: Record<string, unknown>` solto
- Handler reage ao evento, rápido e idempotente (executa na mesma requisição)
- Eventos são in-process: mesma instância, mesma requisição — para integrações que exigem entrega garantida entre processos, use a Facade (ou avalie filas mais tarde)
