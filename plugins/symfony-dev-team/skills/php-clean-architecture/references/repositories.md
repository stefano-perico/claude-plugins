# Repositories - Ports & Adapters - Clean Architecture PHP

## Principe Ports & Adapters

```
Domain (Ports)              Infrastructure (Adapters)
+-------------------+       +------------------------+
| Interface         |  <--  | Implémentation         |
| (contrat)         |       | (technique)            |
+-------------------+       +------------------------+

DeclarantRepositoryInterface  <--  DeclarantRepository (Doctrine)
                              <--  InMemoryDeclarantRepository (Tests)
                              <--  CachedDeclarantRepository (Decorator)
```

## Repository Interface (Port)

Les interfaces sont définies dans le Domain. Elles définissent le **contrat** sans aucun détail technique.

### Interface de Base

```php
<?php

declare(strict_types=1);

namespace App\Domain\Repository;

/**
 * Interface de base pour tous les repositories.
 *
 * @template T of object
 */
interface RepositoryInterface
{
    /**
     * Trouve une entité par son identifiant.
     *
     * @param mixed $id
     * @return T
     * @throws \App\Domain\Exception\NotFound\ResourceNotFoundException
     */
    public function ofId(mixed $id): object;

    /**
     * Persiste une entité.
     *
     * @param T $entity
     */
    public function save(object $entity): void;

    /**
     * Supprime une entité.
     *
     * @param T $entity
     */
    public function remove(object $entity): void;
}
```

### Interface Spécifique

```php
<?php

declare(strict_types=1);

namespace App\Domain\Repository;

use App\Domain\Entity\Declarant;
use App\Domain\ValueObject\AppUuid;

/**
 * @extends RepositoryInterface<Declarant>
 */
interface DeclarantRepositoryInterface extends RepositoryInterface
{
    /**
     * Trouve un déclarant par son UUID.
     *
     * @throws \App\Domain\Exception\NotFound\DeclarantNotFoundException
     */
    public function ofId(mixed $id): Declarant;

    /**
     * Trouve un déclarant par son UUID.
     */
    public function ofUuid(AppUuid $uuid): Declarant;

    /**
     * Trouve un déclarant par son SIREN.
     */
    public function ofSiren(string $siren): ?Declarant;

    /**
     * Retourne un repository filtré (sans les supprimés).
     */
    public function withoutDeleted(): self;

    /**
     * Crée un QueryBuilder pour des requêtes complexes.
     */
    public function createQueryBuilder(): QueryBuilderInterface;

    /**
     * Persiste un déclarant.
     */
    public function save(object $entity): void;
}
```

### Interface QueryBuilder

```php
<?php

declare(strict_types=1);

namespace App\Domain\Repository;

use App\Application\Shared\Dto\PaginatedResult;

/**
 * Interface pour le query builder.
 * Permet de construire des requêtes fluides sans dépendre de Doctrine.
 *
 * @template T of object
 */
interface QueryBuilderInterface
{
    /**
     * @param string $field
     * @param mixed $value
     * @return self<T>
     */
    public function andWhere(string $field, mixed $value): self;

    /**
     * @param string $search
     * @param array<string> $fields
     * @return self<T>
     */
    public function search(string $search, array $fields): self;

    /**
     * @param string $field
     * @param string $order
     * @return self<T>
     */
    public function orderBy(string $field, string $order = 'ASC'): self;

    /**
     * @param int $page
     * @param int $limit
     * @return PaginatedResult<T>
     */
    public function paginate(int $page, int $limit): PaginatedResult;

    /**
     * @return array<T>
     */
    public function getResult(): array;

    /**
     * @return T|null
     */
    public function getOneOrNullResult(): ?object;
}
```

## Repository Doctrine (Adapter)

### DoctrineRepository Abstrait

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Persistence\Doctrine;

use App\Application\Shared\Dto\PaginatedResult;
use App\Domain\Repository\QueryBuilderInterface;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\ORM\QueryBuilder;
use Doctrine\Persistence\ManagerRegistry;

/**
 * @template T of object
 * @extends ServiceEntityRepository<T>
 */
abstract class DoctrineRepository extends ServiceEntityRepository
{
    protected QueryBuilder $qb;

    /**
     * @param class-string<T> $entityClass
     */
    public function __construct(
        ManagerRegistry $registry,
        string $entityClass,
    ) {
        parent::__construct($registry, $entityClass);
        $this->qb = $this->createQueryBuilder('e');
    }

    public function save(object $entity): void
    {
        $this->getEntityManager()->persist($entity);
        $this->getEntityManager()->flush();
    }

    public function remove(object $entity): void
    {
        $this->getEntityManager()->remove($entity);
        $this->getEntityManager()->flush();
    }

    /**
     * @return DoctrineQueryBuilder<T>
     */
    public function createQueryBuilder(string $alias = 'e', ?string $indexBy = null): DoctrineQueryBuilder
    {
        return new DoctrineQueryBuilder(
            parent::createQueryBuilder($alias, $indexBy),
            $this->getEntityManager(),
        );
    }

    protected function cloneWithQueryBuilder(QueryBuilder $qb): static
    {
        $clone = clone $this;
        $clone->qb = $qb;
        return $clone;
    }
}
```

### DoctrineQueryBuilder

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Persistence\Doctrine;

use App\Application\Shared\Dto\PaginatedResult;
use App\Domain\Repository\QueryBuilderInterface;
use Doctrine\ORM\EntityManagerInterface;
use Doctrine\ORM\QueryBuilder;
use Doctrine\ORM\Tools\Pagination\Paginator;

/**
 * @template T of object
 * @implements QueryBuilderInterface<T>
 */
final class DoctrineQueryBuilder implements QueryBuilderInterface
{
    private int $parameterIndex = 0;

    public function __construct(
        private QueryBuilder $qb,
        private EntityManagerInterface $em,
    ) {}

    public function andWhere(string $condition, mixed $value = null): self
    {
        if ($value !== null) {
            $paramName = 'param_' . $this->parameterIndex++;
            $this->qb->andWhere($condition)
                     ->setParameter($paramName, $value);
        } else {
            $this->qb->andWhere($condition);
        }

        return $this;
    }

    public function search(string $search, array $fields): self
    {
        if (empty($fields) || $search === '') {
            return $this;
        }

        $conditions = [];
        $paramName = 'search_' . $this->parameterIndex++;

        foreach ($fields as $field) {
            $conditions[] = sprintf('LOWER(e.%s) LIKE LOWER(:%s)', $field, $paramName);
        }

        $this->qb->andWhere(implode(' OR ', $conditions))
                 ->setParameter($paramName, '%' . $search . '%');

        return $this;
    }

    public function orderBy(string $field, string $order = 'ASC'): self
    {
        $this->qb->orderBy('e.' . $field, $order);
        return $this;
    }

    public function paginate(int $page, int $limit): PaginatedResult
    {
        $this->qb->setFirstResult(($page - 1) * $limit)
                 ->setMaxResults($limit);

        $paginator = new Paginator($this->qb);
        $total = count($paginator);

        return new PaginatedResult(
            items: iterator_to_array($paginator),
            total: $total,
            page: $page,
            limit: $limit,
        );
    }

    public function getResult(): array
    {
        return $this->qb->getQuery()->getResult();
    }

    public function getOneOrNullResult(): ?object
    {
        return $this->qb->getQuery()->getOneOrNullResult();
    }

    public function getQueryBuilder(): QueryBuilder
    {
        return $this->qb;
    }
}
```

### Repository Concret

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Persistence\Doctrine;

use App\Domain\Entity\Declarant;
use App\Domain\Exception\NotFound\DeclarantNotFoundException;
use App\Domain\Repository\DeclarantRepositoryInterface;
use App\Domain\ValueObject\AppUuid;
use Doctrine\Persistence\ManagerRegistry;

/**
 * @extends DoctrineRepository<Declarant>
 */
class DeclarantRepository extends DoctrineRepository implements DeclarantRepositoryInterface
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Declarant::class);
    }

    public function ofId(mixed $id): Declarant
    {
        $entity = $this->find($id);

        if ($entity === null) {
            throw new DeclarantNotFoundException($id);
        }

        return $entity;
    }

    public function ofUuid(AppUuid $uuid): Declarant
    {
        $entity = $this->findOneBy(['uuid' => $uuid]);

        if ($entity === null) {
            throw new DeclarantNotFoundException($uuid);
        }

        return $entity;
    }

    public function ofSiren(string $siren): ?Declarant
    {
        return $this->findOneBy(['siren' => $siren]);
    }

    public function withoutDeleted(): self
    {
        $clone = clone $this;
        $clone->qb = clone $this->qb;
        $clone->qb->andWhere('e.deletedAt IS NULL');

        return $clone;
    }
}
```

## PaginatedResult DTO

```php
<?php

declare(strict_types=1);

namespace App\Application\Shared\Dto;

/**
 * @template T of object
 */
final readonly class PaginatedResult implements \JsonSerializable
{
    /**
     * @param array<T> $items
     */
    public function __construct(
        public array $items,
        public int $total,
        public int $page,
        public int $limit,
    ) {}

    public function getTotalPages(): int
    {
        return (int) ceil($this->total / $this->limit);
    }

    public function hasNextPage(): bool
    {
        return $this->page < $this->getTotalPages();
    }

    public function hasPreviousPage(): bool
    {
        return $this->page > 1;
    }

    public function jsonSerialize(): array
    {
        return [
            'items' => $this->items,
            'meta' => [
                'total' => $this->total,
                'page' => $this->page,
                'limit' => $this->limit,
                'totalPages' => $this->getTotalPages(),
                'hasNextPage' => $this->hasNextPage(),
                'hasPreviousPage' => $this->hasPreviousPage(),
            ],
        ];
    }
}
```

## Filtres Doctrine

### SoftDeleteFilter

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Doctrine\Filter;

use Doctrine\ORM\Mapping\ClassMetadata;
use Doctrine\ORM\Query\Filter\SQLFilter;

class SoftDeleteFilter extends SQLFilter
{
    public function addFilterConstraint(ClassMetadata $targetEntity, $targetTableAlias): string
    {
        if (!$targetEntity->hasField('deletedAt')) {
            return '';
        }

        return sprintf('%s.deleted_at IS NULL', $targetTableAlias);
    }
}
```

### TenantFilter (Multi-tenant)

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Doctrine\Filter;

use Doctrine\ORM\Mapping\ClassMetadata;
use Doctrine\ORM\Query\Filter\SQLFilter;

class TenantFilter extends SQLFilter
{
    public function addFilterConstraint(ClassMetadata $targetEntity, $targetTableAlias): string
    {
        if (!$targetEntity->hasField('tenantId')) {
            return '';
        }

        return sprintf(
            '%s.tenant_id = %s',
            $targetTableAlias,
            $this->getParameter('tenantId')
        );
    }
}
```

## Types Doctrine Custom

### AppUuidType

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Doctrine\Type;

use App\Domain\ValueObject\AppUuid;
use Doctrine\DBAL\Platforms\AbstractPlatform;
use Doctrine\DBAL\Types\Type;

class AppUuidType extends Type
{
    public const NAME = 'app_uuid';

    public function getSQLDeclaration(array $column, AbstractPlatform $platform): string
    {
        return $platform->getGuidTypeDeclarationSQL($column);
    }

    public function convertToPHPValue($value, AbstractPlatform $platform): ?AppUuid
    {
        if ($value === null) {
            return null;
        }

        return AppUuid::fromString($value);
    }

    public function convertToDatabaseValue($value, AbstractPlatform $platform): ?string
    {
        if ($value === null) {
            return null;
        }

        if ($value instanceof AppUuid) {
            return $value->getValue();
        }

        return $value;
    }

    public function getName(): string
    {
        return self::NAME;
    }

    public function requiresSQLCommentHint(AbstractPlatform $platform): bool
    {
        return true;
    }
}
```

### MoneyType

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Doctrine\Type;

use App\Domain\ValueObject\MoneyVO;
use Doctrine\DBAL\Platforms\AbstractPlatform;
use Doctrine\DBAL\Types\Type;

class MoneyType extends Type
{
    public const NAME = 'money';

    public function getSQLDeclaration(array $column, AbstractPlatform $platform): string
    {
        return $platform->getIntegerTypeDeclarationSQL($column);
    }

    public function convertToPHPValue($value, AbstractPlatform $platform): ?MoneyVO
    {
        if ($value === null) {
            return null;
        }

        return new MoneyVO((int) $value);
    }

    public function convertToDatabaseValue($value, AbstractPlatform $platform): ?int
    {
        if ($value === null) {
            return null;
        }

        if ($value instanceof MoneyVO) {
            return $value->getAmount();
        }

        return (int) $value;
    }

    public function getName(): string
    {
        return self::NAME;
    }
}
```

## Repository Cache Decorator

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Persistence\Cache;

use App\Domain\Entity\Declarant;
use App\Domain\Repository\DeclarantRepositoryInterface;
use App\Domain\Repository\QueryBuilderInterface;
use App\Domain\ValueObject\AppUuid;
use Psr\Cache\CacheItemPoolInterface;

final class CachedDeclarantRepository implements DeclarantRepositoryInterface
{
    private const CACHE_TTL = 3600; // 1 heure

    public function __construct(
        private DeclarantRepositoryInterface $decorated,
        private CacheItemPoolInterface $cache,
    ) {}

    public function ofId(mixed $id): Declarant
    {
        $cacheKey = sprintf('declarant_%s', $id);
        $item = $this->cache->getItem($cacheKey);

        if ($item->isHit()) {
            return $item->get();
        }

        $declarant = $this->decorated->ofId($id);

        $item->set($declarant);
        $item->expiresAfter(self::CACHE_TTL);
        $this->cache->save($item);

        return $declarant;
    }

    public function ofUuid(AppUuid $uuid): Declarant
    {
        $cacheKey = sprintf('declarant_uuid_%s', $uuid);
        $item = $this->cache->getItem($cacheKey);

        if ($item->isHit()) {
            return $item->get();
        }

        $declarant = $this->decorated->ofUuid($uuid);

        $item->set($declarant);
        $item->expiresAfter(self::CACHE_TTL);
        $this->cache->save($item);

        return $declarant;
    }

    public function ofSiren(string $siren): ?Declarant
    {
        return $this->decorated->ofSiren($siren);
    }

    public function withoutDeleted(): DeclarantRepositoryInterface
    {
        return new self(
            $this->decorated->withoutDeleted(),
            $this->cache,
        );
    }

    public function createQueryBuilder(): QueryBuilderInterface
    {
        return $this->decorated->createQueryBuilder();
    }

    public function save(object $entity): void
    {
        $this->decorated->save($entity);

        // Invalide le cache
        if ($entity instanceof Declarant) {
            $this->cache->deleteItem(sprintf('declarant_%s', $entity->getId()));
            $this->cache->deleteItem(sprintf('declarant_uuid_%s', $entity->getUuid()));
        }
    }

    public function remove(object $entity): void
    {
        $this->decorated->remove($entity);

        if ($entity instanceof Declarant) {
            $this->cache->deleteItem(sprintf('declarant_%s', $entity->getId()));
            $this->cache->deleteItem(sprintf('declarant_uuid_%s', $entity->getUuid()));
        }
    }
}
```

## InMemory Repository (Tests)

```php
<?php

declare(strict_types=1);

namespace App\Tests\Infrastructure\Persistence;

use App\Domain\Entity\Declarant;
use App\Domain\Exception\NotFound\DeclarantNotFoundException;
use App\Domain\Repository\DeclarantRepositoryInterface;
use App\Domain\Repository\QueryBuilderInterface;
use App\Domain\ValueObject\AppUuid;

final class InMemoryDeclarantRepository implements DeclarantRepositoryInterface
{
    /** @var array<string, Declarant> */
    private array $entities = [];

    private bool $excludeDeleted = false;

    public function ofId(mixed $id): Declarant
    {
        if (!isset($this->entities[$id])) {
            throw new DeclarantNotFoundException($id);
        }

        $entity = $this->entities[$id];

        if ($this->excludeDeleted && $entity->isDeleted()) {
            throw new DeclarantNotFoundException($id);
        }

        return $entity;
    }

    public function ofUuid(AppUuid $uuid): Declarant
    {
        foreach ($this->entities as $entity) {
            if ($entity->getUuid()->equals($uuid)) {
                if ($this->excludeDeleted && $entity->isDeleted()) {
                    throw new DeclarantNotFoundException($uuid);
                }
                return $entity;
            }
        }

        throw new DeclarantNotFoundException($uuid);
    }

    public function ofSiren(string $siren): ?Declarant
    {
        foreach ($this->entities as $entity) {
            if ($entity->getSiren() === $siren) {
                if ($this->excludeDeleted && $entity->isDeleted()) {
                    return null;
                }
                return $entity;
            }
        }

        return null;
    }

    public function withoutDeleted(): self
    {
        $clone = clone $this;
        $clone->excludeDeleted = true;
        return $clone;
    }

    public function createQueryBuilder(): QueryBuilderInterface
    {
        throw new \RuntimeException('Not implemented in InMemory repository');
    }

    public function save(object $entity): void
    {
        if ($entity instanceof Declarant) {
            $key = $entity->getId() ?? spl_object_id($entity);
            $this->entities[$key] = $entity;
        }
    }

    public function remove(object $entity): void
    {
        if ($entity instanceof Declarant) {
            $key = $entity->getId() ?? spl_object_id($entity);
            unset($this->entities[$key]);
        }
    }

    /**
     * Helper pour les tests
     */
    public function add(Declarant $entity): void
    {
        $this->save($entity);
    }

    /**
     * Helper pour les tests
     */
    public function clear(): void
    {
        $this->entities = [];
    }
}
```

## Configuration Services

```yaml
# config/services.yaml
services:
    # Repository interfaces bound to Doctrine implementations
    App\Domain\Repository\DeclarantRepositoryInterface:
        class: App\Infrastructure\Persistence\Doctrine\DeclarantRepository

    # Avec Cache Decorator (optionnel)
    App\Domain\Repository\DeclarantRepositoryInterface:
        class: App\Infrastructure\Persistence\Cache\CachedDeclarantRepository
        arguments:
            $decorated: '@App\Infrastructure\Persistence\Doctrine\DeclarantRepository'
            $cache: '@cache.app'
```

## Checklist Repositories

- [ ] Interface dans `Domain/Repository/`
- [ ] Implémentation dans `Infrastructure/Persistence/Doctrine/`
- [ ] Méthode `ofId()` lance une exception si non trouvé
- [ ] Méthode `save()` persiste ET flush
- [ ] Méthode `withoutDeleted()` retourne un clone filtré
- [ ] QueryBuilder interface pour requêtes complexes
- [ ] Types Doctrine custom pour ValueObjects
- [ ] Tests avec InMemory repository