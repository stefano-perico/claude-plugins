---
name: php-clean-architecture
description: |
  Architecture hexagonale avec CQRS pour PHP 8.3+ et Symfony.
  Utiliser ce skill pour : créer des entités, commands, queries, handlers,
  repositories, value objects, et structurer un projet selon les conventions
  d'architecture propre.
  Compromis accepté : entités Doctrine dans le Domain avec annotations ORM.
  Déclencher quand l'utilisateur demande : création d'entité, CQRS, command,
  query, handler, repository, value object, ou architecture hexagonale.
---

# PHP Clean Architecture Skill

Ce skill guide la création de code PHP suivant une architecture hexagonale avec CQRS,
adaptée aux projets Symfony 6.4+ et PHP 8.3+.

## Principes Fondamentaux

### 1. Architecture Hexagonale (Ports & Adapters)

```
         [Infrastructure]
                |
                v
    +---------------------+
    |                     |
    |  [Application]      |<---- Controllers, CLI
    |       |             |
    |       v             |
    |   [Domain]          |<---- Le coeur métier, sans dépendances externes
    |                     |
    +---------------------+
                ^
                |
         [Infrastructure]
```

**Règles strictes :**
- Le Domain ne dépend de RIEN d'externe (sauf PHP natif)
- L'Application orchestre le Domain via Commands/Queries
- L'Infrastructure implémente les interfaces définies dans Domain

### 2. CQRS (Command Query Responsibility Segregation)

- **Commands** : modifient l'état, ne retournent rien ou l'entité créée/modifiée
- **Queries** : lisent l'état, ne modifient jamais rien
- **Handlers** : fins, délèguent la logique aux Managers/Services

### 3. SOLID

- **S**ingle Responsibility : une classe = une raison de changer
- **O**pen/Closed : ouvert à l'extension, fermé à la modification
- **L**iskov Substitution : les sous-types sont substituables
- **I**nterface Segregation : interfaces spécifiques, pas génériques
- **D**ependency Inversion : dépendre des abstractions (interfaces)

## Structure de Dossiers Obligatoire

```
src/
├── Domain/                          # Couche métier pure
│   ├── Entity/                      # Entités Doctrine
│   ├── ValueObject/                 # Objets valeur immutables
│   ├── Enum/                        # Enums PHP 8.1+
│   ├── Event/                       # Domain events
│   ├── Bus/                         # Interfaces des bus (ports)
│   ├── Repository/                  # Interfaces repositories (ports)
│   ├── Service/                     # Interfaces services (ports)
│   ├── Exception/                   # Organisées par type
│   └── Util/                        # Utilitaires domaine
│
├── Application/                     # Cas d'usage
│   └── [Feature]/                   # Organisé par feature
│       ├── Command/                 # Commands + Handlers
│       ├── Query/                   # Queries + Handlers
│       ├── Manager/                 # Coordinateurs métier
│       ├── Workflow/                # Orchestrateurs complexes
│       └── Service/                 # Implémentations services
│
└── Infrastructure/                  # Adapters techniques
    ├── Persistence/
    │   ├── Doctrine/                # Repositories Doctrine
    │   ├── Api/                     # Repositories API externe
    │   └── Cache/                   # Decorators cache
    ├── Bus/                         # Implémentations bus
    ├── Doctrine/Type/               # Types Doctrine custom
    └── Controller/                  # API endpoints
```

> Voir `references/structure.md` pour les détails complets.

## Conventions de Nommage

### Classes

| Type | Pattern | Exemple |
|------|---------|---------|
| Entity | `{Name}` | `Declarant`, `Invoice` |
| ValueObject | `{Name}VO` | `AdresseVO`, `PersonneVO` |
| Enum | `{Name}` | `Formulaire`, `Status` |
| Command | `{Action}{Entity}Command` | `CreateDeclarantCommand` |
| Query | `{Action}{Entity}Query` | `FindDeclarantItemQuery` |
| Handler | `{Command/Query}Handler` | `CreateDeclarantCommandHandler` |
| Repository Interface | `{Entity}RepositoryInterface` | `DeclarantRepositoryInterface` |
| Repository Impl | `{Entity}Repository` | `DeclarantRepository` |
| Manager | `{Entity}Manager` | `DeclarantManager` |
| Event | `{Entity}{Action}Event` | `DeclarantCreatedEvent` |
| Exception | `{Context}Exception` | `DeclarantNotFoundException` |

### Fichiers et Dossiers

- Un fichier par classe
- Nom du fichier = nom de la classe + `.php`
- Dossiers en PascalCase pour les namespaces

## Patterns de Code

### Command (readonly class)

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\Command;

use App\Domain\Bus\InternalCommandInterface;
use App\Domain\ValueObject\PersonneVO;

final readonly class CreateDeclarantCommand implements InternalCommandInterface
{
    public function __construct(
        public ?string $denomination = null,
        public ?PersonneVO $responsable = null,
    ) {}
}
```

### Handler (thin, délègue)

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

### Query (readonly class)

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

### Repository Interface (Port)

```php
<?php

declare(strict_types=1);

namespace App\Domain\Repository;

use App\Domain\Entity\Declarant;

interface DeclarantRepositoryInterface extends RepositoryInterface
{
    public function ofId(mixed $id): Declarant;

    public function withoutDeleted(): self;

    public function save(Declarant $entity): void;
}
```

### Repository Implementation (Adapter)

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Persistence\Doctrine;

use App\Domain\Entity\Declarant;
use App\Domain\Repository\DeclarantRepositoryInterface;
use App\Infrastructure\Persistence\Doctrine\DoctrineRepository;

class DeclarantRepository extends DoctrineRepository implements DeclarantRepositoryInterface
{
    public function ofId(mixed $id): Declarant
    {
        return $this->find($id) ?? throw new DeclarantNotFoundException($id);
    }

    public function withoutDeleted(): self
    {
        $clone = clone $this;
        $clone->qb->andWhere('e.deletedAt IS NULL');
        return $clone;
    }

    public function save(Declarant $entity): void
    {
        $this->getEntityManager()->persist($entity);
        $this->getEntityManager()->flush();
    }
}
```

> Voir `references/cqrs.md`, `references/entities.md`, et `references/repositories.md` pour tous les patterns.

## Checklist de Validation

Avant de finaliser toute implémentation, vérifier :

### Domain

- [ ] Aucune dépendance vers Application ou Infrastructure
- [ ] Entités avec traits `IdTrait` si applicable
- [ ] ValueObjects immutables (`readonly class`)
- [ ] Enums avec méthodes métier si nécessaire
- [ ] Exceptions organisées par type (NotFound/, Validator/, etc.)

### Application

- [ ] Commands et Queries en `final readonly class`
- [ ] Handlers fins, délèguent aux Managers
- [ ] Un Handler par Command/Query
- [ ] Interfaces Messenger correctes (`InternalCommandInterface`, `InternalQueryInterface`)

### Infrastructure

- [ ] Repositories implémentent les interfaces du Domain
- [ ] Types Doctrine custom pour ValueObjects complexes
- [ ] Pas de logique métier dans les Controllers

### Général

- [ ] `declare(strict_types=1)` en haut de chaque fichier
- [ ] Typage strict partout (paramètres, retours, propriétés)
- [ ] Pas de `mixed` sauf si absolument nécessaire
- [ ] Attributs PHP 8 plutôt qu'annotations

## Génération de Code

Quand l'utilisateur demande de créer une entité avec CQRS complet, générer :

1. **Domain**
   - `Entity/{Name}.php`
   - `Repository/{Name}RepositoryInterface.php`
   - `Exception/NotFound/{Name}NotFoundException.php`

2. **Application/{Name}**
   - `Command/Create{Name}Command.php`
   - `Command/Create{Name}CommandHandler.php`
   - `Command/Update{Name}Command.php`
   - `Command/Update{Name}CommandHandler.php`
   - `Command/Delete{Name}Command.php`
   - `Command/Delete{Name}CommandHandler.php`
   - `Query/Find{Name}ItemQuery.php`
   - `Query/Find{Name}ItemQueryHandler.php`
   - `Query/Find{Name}ListQuery.php`
   - `Query/Find{Name}ListQueryHandler.php`
   - `Manager/{Name}Manager.php`

3. **Infrastructure**
   - `Persistence/Doctrine/{Name}Repository.php`

## Références

Pour les patterns détaillés, consulter :

- `references/structure.md` - Structure de dossiers complète
- `references/cqrs.md` - Patterns CQRS détaillés
- `references/entities.md` - Entités, ValueObjects, Enums, Events
- `references/repositories.md` - Ports et Adapters repositories

## Exemples d'Utilisation

**Utilisateur** : "Crée une entité Invoice avec un montant et une date d'échéance"

**Action** : Générer tous les fichiers CQRS avec :
- Entité `Invoice` avec propriétés `amount` (MoneyVO) et `dueDate` (DateTimeImmutable)
- Repository interface et implementation
- Commands Create/Update/Delete avec handlers
- Queries Find Item/List avec handlers
- Manager avec la logique métier
