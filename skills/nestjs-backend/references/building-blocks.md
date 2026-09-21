# Building Blocks

Componentes fundamentais de um módulo: serviços, entidades e repositórios. Para a estrutura de pastas do módulo, ver `references/architecture-patterns.md`.

## Sumário

1. Serviços (linha ~10)
2. Entidades (linha ~35)
3. Repositório (linha ~55, com Repositório Base obrigatório)

---

## 1. Serviços

Serviços `@Injectable()` em `core/service/` encapsulam a lógica de negócio e interagem com os repositórios via DI — injetando a classe concreta diretamente, sem token Symbol.

```typescript
// src/modules/finance/core/service/wallet.service.ts
import { Injectable } from '@nestjs/common'
import { WalletRepository } from '@modules/finance'
import { Wallet } from './wallet.entity'
import { EVENT_PUBLISHER } from '@common/contracts'

@Injectable()
export class WalletService {
  constructor(
    private readonly walletRepository: WalletRepository,
    @Inject(EVENT_PUBLISHER) private readonly events: EventPublisher,
  ) {}

  async create(userId: string, name: string): Promise<Wallet> {
    const wallet = new Wallet(generateId(), userId, name)
    await this.walletRepository.save(wallet)
    await this.events.publish('finance.wallet.created', { walletId: wallet.id, userId })
    return wallet
  }

  async getById(id: string): Promise<Wallet> {
    const wallet = await this.walletRepository.findById(id)
    if (!wallet) throw new WalletNotFoundError(id)
    return wallet
  }
}
```

---

## 2. Entidades

As entidades de domínio são as **próprias entidades TypeORM** — não há entidade de domínio pura separada. Uma única classe por entidade, em `core/entities/`, com decorators TypeORM e podendo conter métodos de domínio.

```typescript
// src/modules/finance/core/entities/wallet.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, UpdateDateColumn } from 'typeorm'

@Entity('finance_wallets')
export class Wallet {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column()
  userId: string

  @Column()
  name: string

  @CreateDateColumn({ type: 'timestamptz' })
  createdAt: Date

  @UpdateDateColumn({ type: 'timestamptz' })
  updatedAt: Date
}
```

---

## 3. Repositório

Repositórios são **classes concretas** `@Injectable()` em `persistence/repository/` — sem interfaces nem tokens Symbol. O NestJS resolve a injeção pelo tipo da classe.

Todo repositório **estende obrigatoriamente** o `BaseRepository` do `common`, que fornece os métodos genéricos de CRUD (`findById`, `findAll`, `save`, `delete`). O repositório do módulo adiciona apenas o que é específico do domínio.

```typescript
// src/common/typeorm/base.repository.ts
import type { Repository, ObjectLiteral } from 'typeorm'

export abstract class BaseRepository<T extends ObjectLiteral> {
  constructor(protected readonly repository: Repository<T>) {}

  async findById(id: string): Promise<T | null> {
    return this.repository.findOneBy({ id } as any)
  }

  async findAll(): Promise<T[]> {
    return this.repository.find()
  }

  async save(entity: T): Promise<T> {
    return this.repository.save(entity)
  }

  async delete(id: string): Promise<void> {
    await this.repository.delete(id)
  }
}
```

```typescript
// src/modules/finance/persistence/repository/wallet.repository.ts
import { Injectable } from '@nestjs/common'
import { InjectRepository } from '@nestjs/typeorm'
import { Repository } from 'typeorm'
import { BaseRepository } from '@common/typeorm/base.repository'
import { Wallet } from '@modules/finance'

@Injectable()
export class WalletRepository extends BaseRepository<Wallet> {
  constructor(
    @InjectRepository(Wallet)
    repository: Repository<Wallet>,
  ) {
    super(repository)
  }

  async findByUserId(userId: string): Promise<Wallet[]> {
    return this.repository.find({ where: { userId } })
  }

  async countByUser(userId: string): Promise<number> {
    return this.repository.count({ where: { userId } })
  }
}
```
