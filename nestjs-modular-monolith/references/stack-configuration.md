# Configuração da Stack

## Sumário

1. Bootstrap NestJS + Fastify (linha ~10)
2. Setup do Prisma (linha ~60)
3. DTOs e Validação (linha ~100)
4. Exception Filter (linha ~170)
5. Configuração do Biome (linha ~250)
6. Definição de Módulo NestJS (linha ~290)

---

## 1. Bootstrap NestJS + Fastify

Fastify é o adaptador HTTP recomendado para monolitos modulares. É ~2-3x mais rápido que Express, com melhor suporte a TypeScript e uma arquitetura de plugins alinhada com a filosofia modular.

```typescript
// apps/api/src/main.ts
import { NestFactory } from '@nestjs/core'
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify'
import { ValidationPipe, Logger } from '@nestjs/common'
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger'
import { AppModule } from './app.module'
import { HttpExceptionFilter } from '@project/shared/infrastructure'

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter({ logger: true }))

  // Prefixo global
  app.setGlobalPrefix('api')

  // Validação
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
      transformOptions: { enableImplicitConversion: true },
    }),
  )

  // Exception filter
  app.useGlobalFilters(new HttpExceptionFilter())

  // Swagger
  const config = new DocumentBuilder()
    .setTitle('Modular Monolith API')
    .setDescription('Documentação da API')
    .setVersion('1.0')
    .addBearerAuth()
    .build()
  SwaggerModule.setup('docs', app, SwaggerModule.createDocument(app, config))

  const port = process.env.PORT ?? 3000
  await app.listen(port, '0.0.0.0')
  Logger.log(`Aplicação rodando na porta ${port}`, 'Bootstrap')
}

bootstrap()
```

### Alternativa com Express

Se a equipe prefere Express ou precisa de middleware específico do Express:

```typescript
// apps/api/src/main.ts
import { NestFactory } from '@nestjs/core'
import { AppModule } from './app.module'

async function bootstrap() {
  const app = await NestFactory.create(AppModule)
  // ... mesma configuração acima, sem FastifyAdapter
  await app.listen(process.env.PORT ?? 3000)
}

bootstrap()
```

---

## 2. Setup do Prisma

### PrismaService

```typescript
// libs/shared/infrastructure/src/database/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common'
import { PrismaClient } from '@prisma/client'

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(PrismaService.name)

  async onModuleInit() {
    await this.$connect()
    this.logger.log('Conexão com o banco estabelecida')
  }

  async onModuleDestroy() {
    await this.$disconnect()
    this.logger.log('Conexão com o banco encerrada')
  }
}
```

### PrismaModule

```typescript
// libs/shared/infrastructure/src/database/prisma.module.ts
import { Global, Module } from '@nestjs/common'
import { PrismaService } from './prisma.service'

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

---

## 3. DTOs e Validação

### DTO de Criação com Swagger

```typescript
// libs/billing/application/dtos/create-billing-plan.dto.ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger'
import { IsString, IsInt, IsEnum, IsNotEmpty, Min, MinLength, MaxLength } from 'class-validator'
import { Transform } from 'class-transformer'

export class CreateBillingPlanDto {
  @ApiProperty({ example: 'Pro Plan', minLength: 2, maxLength: 100 })
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  @MaxLength(100)
  @Transform(({ value }) => value?.trim())
  name: string

  @ApiProperty({ example: 2999, description: 'Preço em centavos' })
  @IsInt()
  @Min(0)
  priceInCents: number

  @ApiProperty({ enum: ['MONTHLY', 'YEARLY'] })
  @IsEnum(['MONTHLY', 'YEARLY'])
  interval: string
}
```

### DTO de Paginação (Reutilizável)

```typescript
// libs/shared/domain/src/dtos/pagination.dto.ts
import { ApiPropertyOptional } from '@nestjs/swagger'
import { IsOptional, IsString, IsInt, Min, Max } from 'class-validator'
import { Type, Transform } from 'class-transformer'

export class PaginationDto {
  @ApiPropertyOptional({ description: 'Cursor para paginação' })
  @IsOptional()
  @IsString()
  cursor?: string

  @ApiPropertyOptional({ minimum: 1, maximum: 100, default: 20 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  @Transform(({ value }) => value ?? 20)
  limit: number = 20
}
```

### DTO de Resposta

```typescript
// libs/billing/application/dtos/billing-plan-response.dto.ts
import { ApiProperty } from '@nestjs/swagger'
import { BillingPlan } from '../../domain/entities/billing-plan.entity'

export class BillingPlanResponseDto {
  @ApiProperty() id: string
  @ApiProperty() name: string
  @ApiProperty() priceInCents: number
  @ApiProperty() interval: string
  @ApiProperty() createdAt: Date

  static from(plan: BillingPlan): BillingPlanResponseDto {
    const dto = new BillingPlanResponseDto()
    dto.id = plan.id
    dto.name = plan.name
    dto.priceInCents = plan.priceInCents
    dto.interval = plan.interval
    dto.createdAt = plan.createdAt
    return dto
  }
}
```

---

## 4. Exception Filter

Mapeia exceções de domínio para códigos de status HTTP. Mantém a lógica de domínio livre de preocupações HTTP.

```typescript
// libs/shared/infrastructure/src/filters/http-exception.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException, HttpStatus, Logger } from '@nestjs/common'

type DomainErrorClass = new (...args: unknown[]) => Error
type HttpExceptionCreator = (message: string) => HttpException

@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name)

  /** Mapeia exceções de domínio para códigos de status HTTP. Adicione novas exceções de domínio aqui conforme seus módulos crescem. */
  private readonly errorMap = new Map<DomainErrorClass, HttpExceptionCreator>()

  /** Registra um mapeamento de erro de domínio para exceção HTTP. Chame isso na inicialização do módulo para registrar erros específicos do módulo. */
  registerError(errorClass: DomainErrorClass, creator: HttpExceptionCreator): void {
    this.errorMap.set(errorClass, creator)
  }

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp()
    const response = ctx.getResponse()
    const request = ctx.getRequest()

    const httpException = this.resolveHttpException(exception)
    const status = httpException.getStatus()
    const body = httpException.getResponse()

    response.status(status).send({
      ...(typeof body === 'object' ? body : { message: body }),
      timestamp: new Date().toISOString(),
      path: request.url,
    })
  }

  private resolveHttpException(exception: unknown): HttpException {
    if (exception instanceof HttpException) return exception

    if (exception instanceof Error) {
      const mapped = this.mapDomainError(exception)
      if (mapped) return mapped
      this.logger.error(exception.message, exception.stack)
    } else {
      this.logger.error('Exceção desconhecida', exception)
    }

    return new HttpException('Erro interno do servidor', HttpStatus.INTERNAL_SERVER_ERROR)
  }

  private mapDomainError(error: Error): HttpException | null {
    for (const [errorType, creator] of this.errorMap.entries()) {
      if (error instanceof errorType) return creator(error.message)
    }
    return null
  }
}
```

### Uso — Registrando Erros de Módulo

```typescript
// libs/billing/billing.module.ts
import { Module, OnModuleInit } from '@nestjs/common'
import { HttpExceptionFilter } from '@project/shared/infrastructure'
import { NotFoundException, ConflictException } from '@nestjs/common'
import { BillingPlanNotFoundError } from './domain/exceptions'

@Module({
  /* ... */
})
export class BillingModule implements OnModuleInit {
  constructor(private readonly exceptionFilter: HttpExceptionFilter) {}

  onModuleInit() {
    this.exceptionFilter.registerError(BillingPlanNotFoundError, (msg) => new NotFoundException(msg))
  }
}
```

---

## 5. Configuração do Biome

O Biome substitui tanto o ESLint quanto o Prettier por uma única ferramenta significativamente mais rápida.

```json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "style": { "useConst": "error", "useTemplate": "error" },
      "complexity": { "noForEach": "off" },
      "suspicious": { "noExplicitAny": "error" }
    }
  },
  "formatter": {
    "enabled": true,
    "formatWithErrors": false,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100,
    "lineEnding": "lf"
  },
  "javascript": {
    "formatter": { "quoteStyle": "single", "trailingCommas": "all", "semicolons": "always" }
  },
  "files": {
    "include": ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.jsx"],
    "ignore": ["node_modules", "dist", "coverage", ".nx"]
  }
}
```

### Scripts do Package.json

```json
{
  "scripts": {
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "format": "biome format --write ."
  }
}
```

---

## 6. Padrão de Definição de Módulo NestJS

Cada módulo de domínio segue esta estrutura. Este exemplo mostra o **padrão de serviço simples** (default):

```typescript
// libs/billing/billing.module.ts
import { Module } from '@nestjs/common'
import { EventEmitterModule } from '@nestjs/event-emitter'

// Apresentação
import { BillingPlanController } from './presentation/billing-plan.controller'

// Aplicação — Serviços
import { BillingPlanService } from './application/services/billing-plan.service'

// Aplicação — Event Handlers
import { OnUserCreatedHandler } from './application/handlers/on-user-created.handler'

// Infraestrutura — Repositórios
import { PrismaBillingPlanRepository } from './infrastructure/repositories/prisma-billing-plan.repository'

// Domínio — Constantes
import { BILLING_PLAN_REPOSITORY } from './domain/repositories/billing-plan.repository'

// Compartilhado — Eventos
import { EVENT_PUBLISHER } from '@project/shared/contracts'
import { EventPublisherModule } from '@project/shared/infrastructure'

@Module({
  imports: [EventPublisherModule],
  controllers: [BillingPlanController],
  providers: [
    BillingPlanService,
    OnUserCreatedHandler,
    { provide: BILLING_PLAN_REPOSITORY, useClass: PrismaBillingPlanRepository },
  ],
  // Exporte apenas o que outros módulos precisam (contratos, não internals)
  exports: [],
})
export class BillingModule {}
```

Para módulos CQRS, adicione `CqrsModule` aos imports e substitua o serviço por command/query handlers:

```typescript
import { CqrsModule } from '@nestjs/cqrs'

const CommandHandlers = [CreateBillingPlanHandler]
const QueryHandlers = [GetBillingPlanHandler]

@Module({
  imports: [CqrsModule, EventPublisherModule],
  controllers: [BillingPlanController],
  providers: [
    ...CommandHandlers,
    ...QueryHandlers,
    OnUserCreatedHandler,
    { provide: BILLING_PLAN_REPOSITORY, useClass: PrismaBillingPlanRepository },
  ],
  exports: [],
})
export class BillingModule {}
```

**Pontos-chave:**

- Vincule a interface do repositório à implementação via DI
- NÃO exporte nada a menos que outro módulo genuinamente precise
- Comunicação entre módulos vai por eventos, não por exports
- Use serviços simples por default; evolua para CQRS apenas quando a complexidade do módulo justificar
