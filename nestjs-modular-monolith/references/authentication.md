# Autenticação

## Sumário

1. Opção A: Passport + JWT (linha ~15)
2. Opção B: Better Auth (linha ~120)
3. Escolhendo Entre as Opções (linha ~230)

---

## 1. Opção A: Passport + JWT

O padrão de autenticação mais consolidado no ecossistema NestJS. Melhor para equipes já familiarizadas com estratégias do Passport e fluxos JWT padrão.

### Estratégia JWT

```typescript
// libs/identity/infrastructure/strategies/jwt.strategy.ts
import { Injectable } from '@nestjs/common'
import { PassportStrategy } from '@nestjs/passport'
import { ExtractJwt, Strategy } from 'passport-jwt'
import { ConfigService } from '@nestjs/config'

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: config.getOrThrow('JWT_SECRET'),
    })
  }

  async validate(payload: { sub: string; email: string; role: string }) {
    return { userId: payload.sub, email: payload.email, role: payload.role }
  }
}
```

### Auth Guard

```typescript
// libs/shared/infrastructure/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext, UnauthorizedException } from '@nestjs/common'
import { AuthGuard } from '@nestjs/passport'
import { Reflector } from '@nestjs/core'
import { IS_PUBLIC_KEY } from '../decorators/public.decorator'

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super()
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ])
    if (isPublic) return true
    return super.canActivate(context)
  }

  handleRequest(err: unknown, user: unknown) {
    if (err || !user) {
      throw err || new UnauthorizedException('Token inválido ou ausente')
    }
    return user
  }
}
```

### Decorators

```typescript
// libs/shared/infrastructure/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common'

export const IS_PUBLIC_KEY = 'isPublic'
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true)

// libs/shared/infrastructure/decorators/roles.decorator.ts
export const ROLES_KEY = 'roles'
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles)
```

### Roles Guard

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

### Configuração do Módulo de Auth

```typescript
// libs/identity/identity.module.ts
@Module({
  imports: [
    PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.registerAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.getOrThrow('JWT_SECRET'),
        signOptions: { expiresIn: '15m' },
      }),
    }),
  ],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService],
})
export class IdentityModule {}
```

### Aplicar Guards Globalmente

```typescript
// apps/api/src/app.module.ts
@Module({
  providers: [
    { provide: APP_GUARD, useClass: JwtAuthGuard },
    { provide: APP_GUARD, useClass: RolesGuard },
  ],
})
export class AppModule {}
```

### Uso nos Controllers

```typescript
@Controller('billing/plans')
@UseGuards(JwtAuthGuard, RolesGuard)
export class BillingPlanController {
  @Post()
  @Roles('admin')
  create(@Body() dto: CreateBillingPlanDto) {
    /* ... */
  }

  @Get()
  @Public()
  findAll() {
    /* ... */
  }
}
```

---

## 2. Opção B: Better Auth

Uma biblioteca de autenticação moderna e agnóstica de framework para TypeScript com um ecossistema de plugins. Boa para equipes que querem uma solução de auth completa com menos boilerplate. Usa o pacote `@thallesp/nestjs-better-auth` para integração com NestJS.

### Instância do Better Auth

```typescript
// libs/identity/infrastructure/auth/auth.config.ts
import { betterAuth } from 'better-auth'
import { Pool } from 'pg'

export const auth = betterAuth({
  database: new Pool({
    connectionString: process.env.DATABASE_URL,
  }),
  emailAndPassword: {
    enabled: true,
    requireEmailVerification: true,
    minPasswordLength: 8,
    maxPasswordLength: 128,
    sendResetPassword: async ({ user, url, token }) => {
      // Implementar email de redefinição de senha
    },
  },
  emailVerification: {
    sendVerificationEmail: async ({ user, url }) => {
      // Implementar email de verificação
    },
  },
  session: {
    expiresIn: 604800, // 7 dias
    updateAge: 86400, // 1 dia (janela de refresh)
    cookieCache: {
      enabled: true,
      maxAge: 300, // cache de 5 min
    },
  },
})
```

### Provedores Sociais (Opcional)

```typescript
// libs/identity/infrastructure/auth/auth.config.ts
export const auth = betterAuth({
  // ...config base acima
  socialProviders: {
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    },
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
  },
})
```

### Integração com Módulo NestJS

```typescript
// libs/identity/identity.module.ts
import { Module } from '@nestjs/common'
import { AuthModule } from '@thallesp/nestjs-better-auth'
import { auth } from './infrastructure/auth/auth.config'

@Module({
  imports: [AuthModule.forRoot({ auth })],
  exports: [AuthModule],
})
export class IdentityModule {}

// apps/api/src/app.module.ts
import { Module } from '@nestjs/common'
import { IdentityModule } from '@project/identity'

@Module({
  imports: [IdentityModule],
})
export class AppModule {}
```

### Proteção de Rotas com Better Auth

A integração do Better Auth com NestJS registra um `AuthGuard` globalmente por padrão. Todas as rotas são protegidas a menos que explicitamente desabilitado.

```typescript
import { Controller, Get } from '@nestjs/common'
import { Session, UserSession, AllowAnonymous, OptionalAuth } from '@thallesp/nestjs-better-auth'

@Controller('billing/plans')
export class BillingPlanController {
  // Rota protegida — sessão garantida
  @Get('me')
  async getMyPlans(@Session() session: UserSession) {
    return this.planService.findByUser(session.user.id)
  }

  // Rota pública — sem autenticação
  @Get('public')
  @AllowAnonymous()
  async getPublicPlans() {
    return this.planService.findPublic()
  }

  // Auth opcional — sessão pode ser null
  @Get('featured')
  @OptionalAuth()
  async getFeatured(@Session() session: UserSession | null) {
    const plans = await this.planService.findFeatured()
    return { plans, isAuthenticated: !!session }
  }
}
```

### Better Auth com Plugins

O Better Auth suporta plugins para funcionalidades estendidas:

```typescript
import { betterAuth } from 'better-auth'
import { admin, twoFactor } from 'better-auth/plugins'

export const auth = betterAuth({
  // ...config base
  plugins: [
    admin(), // Capacidades de dashboard admin
    twoFactor(), // Autenticação de dois fatores
  ],
})
```

---

## 3. Escolhendo Entre as Opções

| Critério                  | Passport/JWT                                | Better Auth                                          |
| ------------------------- | ------------------------------------------- | ---------------------------------------------------- |
| **Maturidade**            | Testado em batalha, anos em produção        | Mais recente, ecossistema em crescimento             |
| **Integração NestJS**     | Nativa, first-party                         | Adaptador de terceiros (`@thallesp/nestjs-better-auth`) |
| **Complexidade de Setup** | Mais boilerplate, mais controle             | Menos boilerplate, baseado em convenções             |
| **Login Social**          | Via passport-google, passport-github, etc.  | Config de provedores sociais embutida                |
| **Gestão de Sessão**      | Manual (tokens JWT)                         | Embutida com cookie cache                            |
| **2FA / Admin**           | Requer implementação customizada            | Baseado em plugins (uma linha)                       |
| **Lógica Customizada**    | Controle total sobre cada etapa             | Hooks e handlers customizados                        |
| **Banco de Dados**        | Você gerencia tabelas de usuário/sessão     | Gerencia automaticamente as tabelas de auth          |
| **Compatibilidade Fastify** | Requer adaptador `@nestjs/platform-fastify` | Funciona com ambos os adaptadores                   |

**Recomendação:** Use **Passport/JWT** quando precisar de controle total, já tiver infraestrutura de auth existente, ou a equipe já estiver confortável com estratégias do Passport. Use **Better Auth** quando quiser setup rápido, recursos embutidos (login social, 2FA, sessões) e preferir convenção sobre configuração.

### Referência Rápida — Comparação de Decorators

| Ação           | Passport/JWT                | Better Auth                         |
| -------------- | --------------------------- | ----------------------------------- |
| Proteger rota  | `@UseGuards(JwtAuthGuard)`  | Automático (guard global)           |
| Rota pública   | `@Public()`                 | `@AllowAnonymous()`                 |
| Obter usuário  | `@Request() req → req.user` | `@Session() session → session.user` |
| Exigir role    | `@Roles('admin')`           | Guard customizado ou plugin         |
| Auth opcional  | Guard customizado           | `@OptionalAuth()`                   |
