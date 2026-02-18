# Structure de Dossiers - Clean Architecture PHP

## Vue d'Ensemble

```
src/
├── Domain/                          # Couche métier pure (le coeur)
├── Application/                     # Cas d'usage et orchestration
└── Infrastructure/                  # Adapters et implémentations techniques
```

## Domain (Couche Métier)

Le Domain est le coeur de l'application. Il ne dépend de RIEN d'externe.

```
src/Domain/
├── Entity/                          # Entités Doctrine (compromis)
│   ├── Declarant.php
│   ├── Declaration.php
│   └── Traits/
│       ├── IdTrait.php              # id + uuid
│       ├── TimestampableTrait.php   # createdAt, updatedAt
│       └── SoftDeletableTrait.php   # deletedAt
│
├── ValueObject/                     # Objets valeur immutables
│   ├── AdresseVO.php
│   ├── PersonneVO.php
│   ├── AppUuid.php
│   └── MoneyVO.php
│
├── Enum/                            # Enumerations PHP 8.1+
│   ├── Formulaire.php
│   ├── Status.php
│   └── TypeDeclarant.php
│
├── Event/                           # Domain Events
│   ├── AbstractEvent.php
│   ├── DeclarantCreatedEvent.php
│   └── DeclarationSubmittedEvent.php
│
├── Bus/                             # Interfaces des bus (Ports)
│   ├── CommandBusInterface.php
│   ├── QueryBusInterface.php
│   ├── EventBusInterface.php
│   ├── InternalCommandInterface.php
│   └── InternalQueryInterface.php
│
├── Repository/                      # Interfaces repositories (Ports)
│   ├── RepositoryInterface.php      # Interface de base
│   ├── DeclarantRepositoryInterface.php
│   └── DeclarationRepositoryInterface.php
│
├── Service/                         # Interfaces services (Ports)
│   ├── NotificationServiceInterface.php
│   ├── PdfGeneratorInterface.php
│   └── ExternalApiClientInterface.php
│
├── Exception/                       # Exceptions organisées par type
│   ├── DomainException.php          # Base exception
│   ├── NotFound/
│   │   ├── ResourceNotFoundException.php
│   │   ├── DeclarantNotFoundException.php
│   │   └── DeclarationNotFoundException.php
│   ├── Validator/
│   │   ├── ValidationException.php
│   │   └── InvalidStateException.php
│   ├── Security/
│   │   └── AccessDeniedException.php
│   └── BadRequest/
│       └── BusinessRuleException.php
│
└── Util/                            # Utilitaires domaine
    ├── StringUtil.php
    └── DateUtil.php
```

### Règles du Domain

1. **Aucune dépendance externe** (sauf PHP natif et Doctrine annotations)
2. **Interfaces uniquement** pour les services externes
3. **Entités riches** avec logique métier
4. **ValueObjects immutables** (`readonly class`)
5. **Exceptions typées** par contexte HTTP

## Application (Cas d'Usage)

L'Application orchestre le Domain. Organisée par **feature/bounded context**.

```
src/Application/
├── Declarant/                       # Feature Declarant
│   ├── Command/
│   │   ├── CreateDeclarantCommand.php
│   │   ├── CreateDeclarantCommandHandler.php
│   │   ├── UpdateDeclarantCommand.php
│   │   ├── UpdateDeclarantCommandHandler.php
│   │   ├── DeleteDeclarantCommand.php
│   │   └── DeleteDeclarantCommandHandler.php
│   │
│   ├── Query/
│   │   ├── FindDeclarantItemQuery.php
│   │   ├── FindDeclarantItemQueryHandler.php
│   │   ├── FindDeclarantListQuery.php
│   │   └── FindDeclarantListQueryHandler.php
│   │
│   ├── Manager/
│   │   └── DeclarantManager.php     # Logique métier coordinée
│   │
│   ├── Workflow/
│   │   └── DeclarantOnboardingWorkflow.php
│   │
│   └── Service/
│       └── DeclarantValidationService.php
│
├── Declaration/                     # Feature Declaration
│   ├── Command/
│   ├── Query/
│   ├── Manager/
│   └── Workflow/
│
└── Shared/                          # Services partagés
    ├── Service/
    │   └── SlugGenerator.php
    └── Dto/
        └── PaginationDto.php
```

### Règles de l'Application

1. **Organisation par feature** (bounded context)
2. **Commands/Queries en readonly class**
3. **Handlers fins** qui délèguent aux Managers
4. **Managers** contiennent la logique métier complexe
5. **Workflows** pour les processus multi-étapes

## Infrastructure (Adapters)

L'Infrastructure implémente les interfaces du Domain.

```
src/Infrastructure/
├── Persistence/
│   ├── Doctrine/                    # Repositories Doctrine
│   │   ├── DoctrineRepository.php   # Base abstraite
│   │   ├── DeclarantRepository.php
│   │   └── DeclarationRepository.php
│   │
│   ├── Api/                         # Repositories API externe
│   │   └── ExternalDeclarantRepository.php
│   │
│   └── Cache/                       # Decorators cache
│       └── CachedDeclarantRepository.php
│
├── Bus/                             # Implémentations des bus
│   ├── MessengerCommandBus.php
│   ├── MessengerQueryBus.php
│   └── MessengerEventBus.php
│
├── Doctrine/
│   ├── Type/                        # Types Doctrine custom
│   │   ├── AppUuidType.php
│   │   └── MoneyType.php
│   │
│   ├── Filter/                      # Filtres Doctrine
│   │   ├── SoftDeleteFilter.php
│   │   └── TenantFilter.php
│   │
│   └── Listener/                    # Event listeners Doctrine
│       └── TimestampableListener.php
│
├── Service/                         # Implémentations services
│   ├── Notification/
│   │   └── MailerNotificationService.php
│   │
│   ├── Pdf/
│   │   └── DompdfGenerator.php
│   │
│   └── Api/
│       └── HttpExternalApiClient.php
│
├── Controller/                      # API endpoints
│   ├── Api/
│   │   ├── DeclarantController.php
│   │   └── DeclarationController.php
│   │
│   └── Admin/
│       └── DashboardController.php
│
├── Serializer/                      # Normalizers/Denormalizers
│   ├── DeclarantNormalizer.php
│   └── AppUuidNormalizer.php
│
└── Security/                        # Voters, Authenticators
    ├── Voter/
    │   └── DeclarantVoter.php
    └── Authenticator/
        └── ApiTokenAuthenticator.php
```

### Règles de l'Infrastructure

1. **Implémente les interfaces du Domain**
2. **Pas de logique métier** (juste de la plomberie technique)
3. **Controllers fins** qui utilisent le CommandBus/QueryBus
4. **Decorators** pour les préoccupations transversales (cache, logging)

## Configuration Symfony

### services.yaml

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    # Domain - Interfaces auto-wired
    App\Domain\Repository\:
        resource: '../src/Domain/Repository/*Interface.php'

    # Application
    App\Application\:
        resource: '../src/Application/'

    # Infrastructure - Bind interfaces to implementations
    App\Domain\Repository\DeclarantRepositoryInterface:
        class: App\Infrastructure\Persistence\Doctrine\DeclarantRepository

    App\Domain\Bus\CommandBusInterface:
        class: App\Infrastructure\Bus\MessengerCommandBus

    App\Domain\Bus\QueryBusInterface:
        class: App\Infrastructure\Bus\MessengerQueryBus
```

### doctrine.yaml

```yaml
doctrine:
    dbal:
        types:
            app_uuid: App\Infrastructure\Doctrine\Type\AppUuidType
            money: App\Infrastructure\Doctrine\Type\MoneyType

    orm:
        filters:
            soft_delete:
                class: App\Infrastructure\Doctrine\Filter\SoftDeleteFilter
                enabled: true
```

## Namespaces

| Couche | Namespace | Peut dépendre de |
|--------|-----------|------------------|
| Domain | `App\Domain\` | Rien (PHP natif uniquement) |
| Application | `App\Application\` | Domain |
| Infrastructure | `App\Infrastructure\` | Domain, Application, libs externes |

## Anti-Patterns à Éviter

1. **Domain qui dépend de l'Infrastructure**
   ```php
   // MAUVAIS
   use Doctrine\ORM\EntityManagerInterface;
   class DeclarantService {
       public function __construct(private EntityManagerInterface $em) {}
   }
   ```

2. **Logique métier dans les Controllers**
   ```php
   // MAUVAIS
   class DeclarantController {
       public function create(Request $request) {
           // Validation métier ici = MAUVAIS
           if ($request->get('siren') && !$this->isValidSiren($request->get('siren'))) {
               throw new BadRequestException('SIREN invalide');
           }
       }
   }
   ```

3. **Handlers avec logique complexe**
   ```php
   // MAUVAIS - Handler trop gros
   class CreateDeclarantCommandHandler {
       public function __invoke(CreateDeclarantCommand $command): Declarant {
           // 50 lignes de logique métier = MAUVAIS
           // Déléguer au Manager
       }
   }
   ```

4. **Application qui connaît l'Infrastructure**
   ```php
   // MAUVAIS
   use App\Infrastructure\Persistence\Doctrine\DeclarantRepository;
   class DeclarantManager {
       public function __construct(private DeclarantRepository $repo) {}
   }
   // BON
   use App\Domain\Repository\DeclarantRepositoryInterface;
   class DeclarantManager {
       public function __construct(private DeclarantRepositoryInterface $repo) {}
   }
   ```
