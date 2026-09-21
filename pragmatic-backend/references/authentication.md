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

Tokens pertencem ao módulo de identidade e são prefixados com o nome do módulo.

```typescript
// libs/identity/domain/entities/session.ts
export class Session {
  constructor(
    public readonly id: string,
    public readonly userId: string,
    public readonly name: string,
    public readonly tokenHash: string,
    public readonly prefix: string,
    public readonly lastUsedAt: Date | null,
    public readonly expiresAt: Date | null,
    public readonly revokedAt: Date | null,
    public readonly createdAt: Date,
  ) {}

  isActive(now: Date): boolean {
    if (this.revokedAt) return false
    if (this.expiresAt && this.expiresAt <= now) return false
    return true
  }
}
```

```typescript
// libs/identity/infrastructure/entities/session.ts
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
}
```

---

## 2. Geração e Hash do Token

O token **nunca é armazenado em texto puro** — apenas seu hash SHA-256. O usuário vê o token completo uma única vez, no momento da criação.

```typescript
// libs/identity/infrastructure/crypto/token-generator.ts
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
// libs/identity/application/services/session.service.ts
import { Inject } from '@nestjs/common'

export const IDENTITY_SESSION_REPOSITORY = Symbol('SessionRepository')

export interface SessionRepository {
  findByTokenHash(hash: string): Promise<Session | null>
  findByUserId(userId: string): Promise<Session[]>
  save(token: Session): Promise<Session>
  delete(id: string): Promise<void>
}

@Injectable()
export class SessionService {
  constructor(
    @Inject(IDENTITY_SESSION_REPOSITORY)
    private readonly repository: SessionRepository,
  ) {}

  async create(userId: string, name: string, expiresInDays?: number): Promise<string> {
    const { token, hash, prefix } = generateSessionToken()
    const expiresAt = expiresInDays
      ? new Date(Date.now() + expiresInDays * 24 * 60 * 60 * 1000)
      : null
    await this.repository.save(
      new Session(generateId(), userId, name, hash, prefix, null, expiresAt, null, new Date()),
    )
    return token // retornado UMA única vez
  }

  async authenticate(token: string): Promise<Session | null> {
    const record = await this.repository.findByTokenHash(hashToken(token))
    if (!record || !record.isActive(new Date())) return null
    await this.repository.save(
      new Session(
        record.id, record.userId, record.name, record.tokenHash, record.prefix,
        new Date(), record.expiresAt, record.revokedAt, record.createdAt,
      ),
    )
    return record
  }

  async revoke(userId: string, tokenId: string): Promise<void> {
    const tokens = await this.repository.findByUserId(userId)
    const token = tokens.find((t) => t.id === tokenId)
    if (!token) throw new SessionNotFoundError(tokenId)
    await this.repository.save(
      new Session(
        token.id, token.userId, token.name, token.tokenHash, token.prefix,
        token.lastUsedAt, token.expiresAt, new Date(), token.createdAt,
      ),
    )
  }

  async list(userId: string): Promise<Session[]> {
    return this.repository.findByUserId(userId)
  }
}
```

---

## 4. Auth Guard

Guard nativo NestJS — sem Passport. Extrai o `Bearer` token, valida contra o banco e anexa o usuário à requisição.

```typescript
// libs/shared/infrastructure/guards/session-auth.guard.ts
import { Injectable, CanActivate, ExecutionContext, UnauthorizedException } from '@nestjs/common'
import { Reflector } from '@nestjs/core'
import { IS_PUBLIC_KEY } from '../decorators/public.decorator'
import { SessionService } from '@project/identity'

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
// libs/shared/infrastructure/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common'

export const IS_PUBLIC_KEY = 'isPublic'
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true)

// libs/shared/infrastructure/decorators/roles.decorator.ts
export const ROLES_KEY = 'roles'
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles)
```

---

## 6. Roles Guard

```typescript
// libs/shared/infrastructure/guards/roles.guard.ts
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

A interface do repositório vive no domínio; a implementação TypeORM na infraestrutura, mapeando a entidade TypeORM `Session` (entidades/session) para a entidade de domínio `Session` (domain/entities), ambas sem sufixo.

```typescript
// libs/identity/infrastructure/repositories/typeorm-session.repository.ts
import { Injectable } from '@nestjs/common'
import { InjectRepository } from '@nestjs/typeorm'
import { IsNull, Repository } from 'typeorm'
import { Session as SessionDomain } from '../../domain/entities/session'
import { IDENTITY_SESSION_REPOSITORY, SessionRepository } from '../../application/services/session.service'
import { Session } from '../entities/session'

@Injectable()
export class TypeOrmSessionRepository implements SessionRepository {
  constructor(
    @InjectRepository(Session)
    private readonly repository: Repository<Session>,
  ) {}

  async findByTokenHash(hash: string): Promise<SessionDomain | null> {
    const data = await this.repository.findOneBy({ tokenHash: hash })
    return data ? this.toDomain(data) : null
  }

  async findByUserId(userId: string): Promise<SessionDomain[]> {
    const data = await this.repository.find({ where: { userId }, order: { createdAt: 'DESC' } })
    return data.map((row) => this.toDomain(row))
  }

  async save(session: SessionDomain): Promise<SessionDomain> {
    await this.repository.save({
      id: session.id,
      userId: session.userId,
      name: session.name,
      tokenHash: session.tokenHash,
      prefix: session.prefix,
      lastUsedAt: session.lastUsedAt,
      expiresAt: session.expiresAt,
      revokedAt: session.revokedAt,
      createdAt: session.createdAt,
    })
    return session
  }

  async delete(id: string): Promise<void> {
    await this.repository.delete(id)
  }

  private toDomain(data: Session): SessionDomain {
    return new SessionDomain(
      data.id,
      data.userId,
      data.name,
      data.tokenHash,
      data.prefix,
      data.lastUsedAt,
      data.expiresAt,
      data.revokedAt,
      data.createdAt,
    )
  }
}
```

---

## 8. Configuração do Módulo de Auth

```typescript
// libs/identity/identity.module.ts
import { TypeOrmModule } from '@nestjs/typeorm'
import { Session } from './infrastructure/entities/session'

@Module({
  imports: [TypeOrmModule.forFeature([Session])],
  providers: [
    SessionService,
    { provide: IDENTITY_SESSION_REPOSITORY, useClass: TypeOrmSessionRepository },
  ],
  exports: [SessionService],
})
export class IdentityModule {}
```

---

## 9. Aplicar Guards Globalmente

```typescript
// apps/api/src/app.module.ts
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
