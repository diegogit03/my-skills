# Comunicação entre Módulos

## Sumário

1. Public API — Comunicação Síncrona (linha ~15)

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
