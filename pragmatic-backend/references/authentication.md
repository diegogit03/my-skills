# Autenticação

O padrão recomendado é **Personal Access Token (PAT) opaco** — tokens aleatórios, prefixados e revogáveis, armazenados com hash no banco. Sem dependência de Passport nem de assinatura JWT: o token é a própria credencial, validado por lookup no banco.

## Sumário

1. Entidade e Tabela de Tokens (linha ~10)
2. Geração e Hash do Token (linha ~65)
3. Token Service (linha ~105)
4. Auth Guard (linha ~155)
5. Decorators (linha ~205)
6. Roles Guard (linha ~220)
7. Implementação do Repositório com TypeORM (linha ~240)
8. Configuração do Módulo de Auth (linha ~275)
9. Aplicar Guards Globalmente (linha ~295)
10. Uso nos Controllers (linha ~310)
11. Rotação e Revogação (linha ~340)

---

## 1. Entidade e Tabela de Tokens

Tokens pertencem ao módulo de identidade e são prefixados com o nome do módulo. A entidade de domínio é a **própria entidade TypeORM** — não há entidade de domínio pura separada; métodos de domínio (como `isActive`) vivem na própria entidade.

```typescript
// src/modules/identity/core/entities/session.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, Index } from 'typeorm'

@Entity('sessions')
@Index('IDX_sessions_user_id', ['userId'])
export class Session {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column()
  userId: string

  @Column()
  name: string

  @Column({ type: 'varchar', unique: true })
  tokenHash: string

  @Column()
  prefix: string

  @Column({ type: 'timestamptz', nullable: true })
  lastUsedAt: Date | null

  @Column({ type: 'timestamptz', nullable: true })
  expiresAt: Date | null

  @Column({ type: 'timestamptz', nullable: true })
  revokedAt: Date | null

  @CreateDateColumn({ type: 'timestamptz' })
  createdAt: Date

  isActive(now: Date): boolean {
    if (this.revokedAt) return false
    if (this.expiresAt && this.expiresAt <= now) return false
    return true
  }
}
```

---

## 2. Geração e Hash do Token

O token **nunca é armazenado em texto puro** — apenas seu hash SHA-256. O usuário vê o token completo uma única vez, no momento da criação.

```typescript
// src/modules/identity/core/crypto/token-generator.ts
import { randomBytes, createHash } from 'crypto'

const PREFIX = 'pat'
const TOKEN_BYTES = 32

export function generateSessionToken(): { token: string; hash: string; prefix: string } {
  const secret = randomBytes(TOKEN_BYTES).toString('base64url')
  const token = `${PREFIX}_${secret}`
  return {
    token,
    hash: hashToken(token),
    prefix: token.slice(0, 12), // exibido na UI: "pat_a1b2c3d4…"
  }
}

export function hashToken(token: string): string {
  return createHash('sha256').update(token).digest('hex')
}
```

---

## 3. Token Service

```typescript
// src/modules/identity/core/service/session.service.ts
import { Injectable } from '@nestjs/common'
import { SessionRepository } from '../../persistence/repository/session.repository'

@Injectable()
export class SessionService {
  constructor(
    private readonly sessionRepository: SessionRepository,
  ) {}

  async create(userId: string, name: string, expiresInDays?: number): Promise<string> {
    const { token, hash, prefix } = generateSessionToken()
    const expiresAt = expiresInDays
      ? new Date(Date.now() + expiresInDays * 24 * 60 * 60 * 1000)
      : null
    const session = new Session()
    session.id = generateId()
    session.userId = userId
    session.name = name
    session.tokenHash = hash
    session.prefix = prefix
    session.expiresAt = expiresAt
    await this.sessionRepository.save(session)
    return token // retornado UMA única vez
  }

  async authenticate(token: string): Promise<Session | null> {
    const record = await this.sessionRepository.findByTokenHash(hashToken(token))
    if (!record || !record.isActive(new Date())) return null
    record.lastUsedAt = new Date()
    await this.sessionRepository.save(record)
    return record
  }

  async revoke(userId: string, tokenId: string): Promise<void> {
    const tokens = await this.sessionRepository.findByUserId(userId)
    const token = tokens.find((t) => t.id === tokenId)
    if (!token) throw new SessionNotFoundError(tokenId)
    token.revokedAt = new Date()
    await this.sessionRepository.save(token)
  }

  async list(userId: string): Promise<Session[]> {
    return this.sessionRepository.findByUserId(userId)
  }
}
```

---

## 4. Auth Guard

Guard nativo NestJS — sem Passport. Extrai o `Bearer` token, valida contra o banco e anexa o usuário à requisição.

```typescript
// src/common/infrastructure/guards/session-auth.guard.ts
import { Injectable, CanActivate, ExecutionContext, UnauthorizedException } from '@nestjs/common'
import { Reflector } from '@nestjs/core'
import { IS_PUBLIC_KEY } from '../decorators/public.decorator'
import { SessionService } from '@modules/identity'

@Injectable()
export class SessionAuthGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly sessionService: SessionService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ])
    if (isPublic) return true

    const request = context.switchToHttp().getRequest()
    const authHeader = request.headers['authorization']
    if (!authHeader?.startsWith('Bearer ')) {
      throw new UnauthorizedException('Token inválido ou ausente')
    }

    const token = authHeader.slice('Bearer '.length).trim()
    const record = await this.tokenService.authenticate(token)
    if (!record) {
      throw new UnauthorizedException('Token inválido ou ausente')
    }

    request.user = { userId: record.userId, tokenId: record.id }
    return true
  }
}
```

---

## 5. Decorators

```typescript
// src/common/infrastructure/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common'

export const IS_PUBLIC_KEY = 'isPublic'
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true)

// src/common/infrastructure/decorators/roles.decorator.ts
export const ROLES_KEY = 'roles'
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles)
```

---

## 6. Roles Guard

```typescript
// src/common/infrastructure/guards/roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ])
    if (!requiredRoles) return true

    const { user } = context.switchToHttp().getRequest()
    return requiredRoles.includes(user.role)
  }
}
```

---

## 7. Implementação do Repositório com TypeORM

Repositório como **classe concreta** `@Injectable()` em `persistence/repository/` — sem interface nem token Symbol. Trabalha diretamente com a entidade TypeORM `Session` (core/entities/session.entity.ts), que também é a entidade de domínio.

```typescript
// src/modules/identity/persistence/repository/session.repository.ts
import { Injectable } from '@nestjs/common'
import { InjectRepository } from '@nestjs/typeorm'
import { Repository } from 'typeorm'
import { Session } from '../../core/entities/session.entity'

@Injectable()
export class SessionRepository {
  constructor(
    @InjectRepository(Session)
    private readonly repository: Repository<Session>,
  ) {}

  async findByTokenHash(hash: string): Promise<Session | null> {
    return this.repository.findOneBy({ tokenHash: hash })
  }

  async findByUserId(userId: string): Promise<Session[]> {
    return this.repository.find({ where: { userId }, order: { createdAt: 'DESC' } })
  }

  async save(session: Session): Promise<Session> {
    return this.repository.save(session)
  }

  async delete(id: string): Promise<void> {
    await this.repository.delete(id)
  }
}
```

---

## 8. Configuração do Módulo de Auth

```typescript
// src/modules/identity/identity.module.ts
import { TypeOrmModule } from '@nestjs/typeorm'
import { Session } from './core/entities/session.entity'

@Module({
  imports: [TypeOrmModule.forFeature([Session])],
  providers: [
    SessionService,
    SessionRepository,
  ],
  exports: [SessionService],
})
export class IdentityModule {}
```

---

## 9. Aplicar Guards Globalmente

```typescript
// src/app.module.ts
@Module({
  providers: [
    { provide: APP_GUARD, useClass: SessionAuthGuard },
    { provide: APP_GUARD, useClass: RolesGuard },
  ],
})
export class AppModule {}
```

---

## 10. Uso nos Controllers

```typescript
@Controller('identity/sessions')
@UseGuards(SessionAuthGuard)
export class SessionController {
  constructor(private readonly sessionService: SessionService) {}

  // Cria um token — o valor completo é retornado UMA única vez
  @Post()
  async create(@Request() req, @Body() dto: CreateSessionDto) {
    const token = await this.sessionService.create(req.user.userId, dto.name, dto.expiresInDays)
    return { token, warning: 'Guarde este token — ele não será exibido novamente' }
  }

  @Get()
  async list(@Request() req) {
    const tokens = await this.sessionService.list(req.user.userId)
    // Nunca retorne o hash; apenas prefixo e metadados
    return tokens.map(({ id, name, prefix, lastUsedAt, expiresAt, revokedAt, createdAt }) => ({
      id, name, prefix, lastUsedAt, expiresAt, revokedAt, createdAt,
    }))
  }

  @Delete(':id')
  async revoke(@Request() req, @Param('id') id: string) {
    await this.sessionService.revoke(req.user.userId, id)
    return { revoked: true }
  }
}
```

---

## 11. Rotação e Revogação

- **Revogação:** sempre explícita (DELETE) ou via expiração. Tokens revogados falham imediatamente no lookup.
- **Rotação:** crie um novo token, valide o uso do novo, depois revogue o antigo. Não há janela de sobreposição automática.
- **Exposição acidental:** se um token vazar, revogue-o imediatamente e emita outro. O hash no banco não ajuda a "desfazer" a exposição.

## Referência Rápida — Decorators

| Ação           | Decorator                           |
| -------------- | ----------------------------------- |
| Proteger rota  | `@UseGuards(SessionAuthGuard)`|
| Rota pública   | `@Public()`                         |
| Obter usuário  | `@Request() req → req.user`         |
| Exigir role    | `@Roles('admin')`                   |

## Decisões de Segurança

- **Hash SHA-256, não bcrypt:** o token é aleatório de 32 bytes (entropia alta), então lookup por hash é seguro e O(1). Bcrypt seria desnecessariamente lento para uma validação por requisição.
- **Sem assinatura:** diferente de JWT, nada é assinado — a validade vem do registro no banco. Isso torna o token revogável instantaneamente.
- **`lastUsedAt` atualizado a cada uso** para auditoria de tokens esquecidos/órfãos.
