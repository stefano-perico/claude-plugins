# Patterns CQRS - Clean Architecture PHP

## Principes CQRS

**Command Query Responsibility Segregation** sépare les opérations de lecture et d'écriture :

- **Commands** : modifient l'état du système
- **Queries** : lisent l'état sans le modifier
- **Handlers** : exécutent les Commands/Queries

## Interfaces Bus (Domain)

### InternalCommandInterface

```php
<?php

declare(strict_types=1);

namespace App\Domain\Bus;

/**
 * Marker interface pour les commands internes.
 * Toutes les commands doivent implémenter cette interface.
 */
interface InternalCommandInterface
{
}
```

### InternalQueryInterface

```php
<?php

declare(strict_types=1);

namespace App\Domain\Bus;

/**
 * Marker interface pour les queries internes.
 * Toutes les queries doivent implémenter cette interface.
 */
interface InternalQueryInterface
{
}
```

### CommandBusInterface

```php
<?php

declare(strict_types=1);

namespace App\Domain\Bus;

interface CommandBusInterface
{
    /**
     * Dispatch une command et retourne le résultat.
     *
     * @template T
     * @param InternalCommandInterface $command
     * @return T|null
     */
    public function dispatch(InternalCommandInterface $command): mixed;
}
```

### QueryBusInterface

```php
<?php

declare(strict_types=1);

namespace App\Domain\Bus;

interface QueryBusInterface
{
    /**
     * Dispatch une query et retourne le résultat.
     *
     * @template T
     * @param InternalQueryInterface $query
     * @return T
     */
    public function dispatch(InternalQueryInterface $query): mixed;
}
```

## Commands

### Structure d'une Command

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Command;

use App\Domain\Bus\InternalCommandInterface;
use App\Domain\ValueObject\PersonneVO;
use App\Domain\ValueObject\AdresseVO;
use App\Domain\Enum\TypeDeclarant;

/**
 * Command pour créer un déclarant.
 *
 * Propriétés nullable = optionnelles.
 * Valeurs par défaut dans le constructeur.
 */
final readonly class CreateDeclarantCommand implements InternalCommandInterface
{
    public function __construct(
        public ?string $denomination = null,
        public ?string $siren = null,
        public ?TypeDeclarant $type = null,
        public ?PersonneVO $responsable = null,
        public ?AdresseVO $adresse = null,
    ) {}
}
```

### Command de Mise à Jour

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Command;

use App\Domain\Bus\InternalCommandInterface;
use App\Domain\ValueObject\AppUuid;
use App\Domain\ValueObject\PersonneVO;

final readonly class UpdateDeclarantCommand implements InternalCommandInterface
{
    public function __construct(
        public AppUuid $id,                    // Requis pour identifier l'entité
        public ?string $denomination = null,
        public ?PersonneVO $responsable = null,
    ) {}
}
```

### Command de Suppression

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Command;

use App\Domain\Bus\InternalCommandInterface;
use App\Domain\ValueObject\AppUuid;

final readonly class DeleteDeclarantCommand implements InternalCommandInterface
{
    public function __construct(
        public AppUuid $id,
    ) {}
}
```

### Command avec Collection

```php
<?php

declare(strict_types=1);

namespace App\Application\Declaration\Command;

use App\Domain\Bus\InternalCommandInterface;
use App\Domain\ValueObject\AppUuid;

final readonly class AddLignesDeclarationCommand implements InternalCommandInterface
{
    /**
     * @param AppUuid $declarationId
     * @param array<LigneData> $lignes
     */
    public function __construct(
        public AppUuid $declarationId,
        public array $lignes = [],
    ) {}
}
```

## Queries

### Query Item (une entité)

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Query;

use App\Domain\Bus\InternalQueryInterface;
use App\Domain\ValueObject\AppUuid;

final readonly class FindDeclarantItemQuery implements InternalQueryInterface
{
    public function __construct(
        public AppUuid $id,
    ) {}
}
```

### Query List (collection paginée)

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Query;

use App\Domain\Bus\InternalQueryInterface;
use App\Domain\Enum\TypeDeclarant;

final readonly class FindDeclarantListQuery implements InternalQueryInterface
{
    public function __construct(
        public ?string $search = null,
        public ?TypeDeclarant $type = null,
        public int $page = 1,
        public int $limit = 20,
        public string $sortBy = 'createdAt',
        public string $sortOrder = 'DESC',
    ) {}
}
```

### Query avec Filtres Complexes

```php
<?php

declare(strict_types=1);

namespace App\Application\Declaration\Query;

use App\Domain\Bus\InternalQueryInterface;
use App\Domain\ValueObject\AppUuid;
use App\Domain\Enum\Status;

final readonly class FindDeclarationListQuery implements InternalQueryInterface
{
    /**
     * @param array<Status>|null $statuses
     */
    public function __construct(
        public ?AppUuid $declarantId = null,
        public ?array $statuses = null,
        public ?\DateTimeImmutable $dateFrom = null,
        public ?\DateTimeImmutable $dateTo = null,
        public int $page = 1,
        public int $limit = 20,
    ) {}
}
```

## Handlers

### Command Handler (Create)

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Command;

use App\Application\Declarant\Manager\DeclarantManager;
use App\Domain\Entity\Declarant;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final readonly class CreateDeclarantCommandHandler
{
    public function __construct(
        private DeclarantManager $manager,
    ) {}

    public function __invoke(CreateDeclarantCommand $command): Declarant
    {
        return $this->manager->create($command);
    }
}
```

### Command Handler (Update)

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Command;

use App\Application\Declarant\Manager\DeclarantManager;
use App\Domain\Entity\Declarant;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final readonly class UpdateDeclarantCommandHandler
{
    public function __construct(
        private DeclarantManager $manager,
    ) {}

    public function __invoke(UpdateDeclarantCommand $command): Declarant
    {
        return $this->manager->update($command);
    }
}
```

### Command Handler (Delete)

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Command;

use App\Application\Declarant\Manager\DeclarantManager;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final readonly class DeleteDeclarantCommandHandler
{
    public function __construct(
        private DeclarantManager $manager,
    ) {}

    public function __invoke(DeleteDeclarantCommand $command): void
    {
        $this->manager->delete($command);
    }
}
```

### Query Handler (Item)

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Query;

use App\Domain\Entity\Declarant;
use App\Domain\Repository\DeclarantRepositoryInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final readonly class FindDeclarantItemQueryHandler
{
    public function __construct(
        private DeclarantRepositoryInterface $repository,
    ) {}

    public function __invoke(FindDeclarantItemQuery $query): Declarant
    {
        return $this->repository->ofId($query->id);
    }
}
```

### Query Handler (List avec Pagination)

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Query;

use App\Application\Shared\Dto\PaginatedResult;
use App\Domain\Repository\DeclarantRepositoryInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final readonly class FindDeclarantListQueryHandler
{
    public function __construct(
        private DeclarantRepositoryInterface $repository,
    ) {}

    public function __invoke(FindDeclarantListQuery $query): PaginatedResult
    {
        $qb = $this->repository
            ->withoutDeleted()
            ->createQueryBuilder();

        if ($query->search !== null) {
            $qb->search($query->search, ['denomination', 'siren']);
        }

        if ($query->type !== null) {
            $qb->andWhere('e.type = :type')
               ->setParameter('type', $query->type);
        }

        return $qb
            ->orderBy($query->sortBy, $query->sortOrder)
            ->paginate($query->page, $query->limit);
    }
}
```

## Managers

Le Manager contient la logique métier complexe. Les Handlers délèguent au Manager.

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Manager;

use App\Application\Declarant\Command\CreateDeclarantCommand;
use App\Application\Declarant\Command\UpdateDeclarantCommand;
use App\Application\Declarant\Command\DeleteDeclarantCommand;
use App\Domain\Entity\Declarant;
use App\Domain\Event\DeclarantCreatedEvent;
use App\Domain\Repository\DeclarantRepositoryInterface;
use App\Domain\Bus\EventBusInterface;

final readonly class DeclarantManager
{
    public function __construct(
        private DeclarantRepositoryInterface $repository,
        private EventBusInterface $eventBus,
    ) {}

    public function create(CreateDeclarantCommand $command): Declarant
    {
        $declarant = new Declarant();

        $declarant->setDenomination($command->denomination);
        $declarant->setSiren($command->siren);
        $declarant->setType($command->type);
        $declarant->setResponsable($command->responsable);
        $declarant->setAdresse($command->adresse);

        $this->repository->save($declarant);

        $this->eventBus->dispatch(new DeclarantCreatedEvent($declarant->getUuid()));

        return $declarant;
    }

    public function update(UpdateDeclarantCommand $command): Declarant
    {
        $declarant = $this->repository->ofId($command->id);

        if ($command->denomination !== null) {
            $declarant->setDenomination($command->denomination);
        }

        if ($command->responsable !== null) {
            $declarant->setResponsable($command->responsable);
        }

        $this->repository->save($declarant);

        return $declarant;
    }

    public function delete(DeleteDeclarantCommand $command): void
    {
        $declarant = $this->repository->ofId($command->id);

        // Soft delete
        $declarant->setDeletedAt(new \DateTimeImmutable());

        $this->repository->save($declarant);
    }
}
```

## Bus Implementations (Infrastructure)

### MessengerCommandBus

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Bus;

use App\Domain\Bus\CommandBusInterface;
use App\Domain\Bus\InternalCommandInterface;
use Symfony\Component\Messenger\MessageBusInterface;
use Symfony\Component\Messenger\Stamp\HandledStamp;

final readonly class MessengerCommandBus implements CommandBusInterface
{
    public function __construct(
        private MessageBusInterface $commandBus,
    ) {}

    public function dispatch(InternalCommandInterface $command): mixed
    {
        $envelope = $this->commandBus->dispatch($command);
        $handledStamp = $envelope->last(HandledStamp::class);

        return $handledStamp?->getResult();
    }
}
```

### MessengerQueryBus

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Bus;

use App\Domain\Bus\QueryBusInterface;
use App\Domain\Bus\InternalQueryInterface;
use Symfony\Component\Messenger\MessageBusInterface;
use Symfony\Component\Messenger\Stamp\HandledStamp;

final readonly class MessengerQueryBus implements QueryBusInterface
{
    public function __construct(
        private MessageBusInterface $queryBus,
    ) {}

    public function dispatch(InternalQueryInterface $query): mixed
    {
        $envelope = $this->queryBus->dispatch($query);
        $handledStamp = $envelope->last(HandledStamp::class);

        return $handledStamp?->getResult();
    }
}
```

## Controller Example

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Controller\Api;

use App\Application\Declarant\Command\CreateDeclarantCommand;
use App\Application\Declarant\Query\FindDeclarantItemQuery;
use App\Application\Declarant\Query\FindDeclarantListQuery;
use App\Domain\Bus\CommandBusInterface;
use App\Domain\Bus\QueryBusInterface;
use App\Domain\ValueObject\AppUuid;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/api/declarants')]
final class DeclarantController extends AbstractController
{
    public function __construct(
        private readonly CommandBusInterface $commandBus,
        private readonly QueryBusInterface $queryBus,
    ) {}

    #[Route('', methods: ['GET'])]
    public function list(Request $request): JsonResponse
    {
        $result = $this->queryBus->dispatch(new FindDeclarantListQuery(
            search: $request->query->get('search'),
            page: $request->query->getInt('page', 1),
            limit: $request->query->getInt('limit', 20),
        ));

        return $this->json($result);
    }

    #[Route('/{id}', methods: ['GET'])]
    public function item(string $id): JsonResponse
    {
        $declarant = $this->queryBus->dispatch(new FindDeclarantItemQuery(
            id: AppUuid::fromString($id),
        ));

        return $this->json($declarant);
    }

    #[Route('', methods: ['POST'])]
    public function create(Request $request): JsonResponse
    {
        $data = $request->toArray();

        $declarant = $this->commandBus->dispatch(new CreateDeclarantCommand(
            denomination: $data['denomination'] ?? null,
            siren: $data['siren'] ?? null,
        ));

        return $this->json($declarant, Response::HTTP_CREATED);
    }
}
```

## Configuration Messenger

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        default_bus: command.bus

        buses:
            command.bus:
                middleware:
                    - doctrine_transaction

            query.bus: ~

            event.bus:
                default_middleware: allow_no_handlers

        routing:
            'App\Domain\Bus\InternalCommandInterface': command.bus
            'App\Domain\Bus\InternalQueryInterface': query.bus
```

## Checklist CQRS

- [ ] Commands en `final readonly class`
- [ ] Queries en `final readonly class`
- [ ] Handlers avec `#[AsMessageHandler]`
- [ ] Handlers fins (< 10 lignes)
- [ ] Logique métier dans les Managers
- [ ] Bus interfaces dans Domain
- [ ] Bus implementations dans Infrastructure
- [ ] Controllers utilisent CommandBus/QueryBus