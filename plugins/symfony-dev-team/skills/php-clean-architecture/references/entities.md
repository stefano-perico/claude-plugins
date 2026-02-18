# Entités, ValueObjects, Enums, Events - Clean Architecture PHP

## Entités Doctrine

### Compromis Accepté

Les entités Doctrine sont placées dans le Domain avec les annotations ORM.
C'est un compromis pragmatique qui permet de garder la simplicité tout en maintenant
la séparation des couches.

### Entity de Base avec Traits

```php
<?php

declare(strict_types=1);

namespace App\Domain\Entity;

use App\Domain\Entity\Traits\IdTrait;
use App\Domain\Entity\Traits\TimestampableTrait;
use App\Domain\Entity\Traits\SoftDeletableTrait;
use App\Domain\ValueObject\AppUuid;
use App\Domain\ValueObject\PersonneVO;
use App\Domain\ValueObject\AdresseVO;
use App\Domain\Enum\TypeDeclarant;
use App\Infrastructure\Doctrine\Filter\DeclarantFilter;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\Table(name: 'declarant')]
#[ORM\HasLifecycleCallbacks]
#[AccessControlled(filters: [DeclarantFilter::class])]
class Declarant
{
    use IdTrait;
    use TimestampableTrait;
    use SoftDeletableTrait;

    #[ORM\Column(type: 'app_uuid', unique: true)]
    private readonly AppUuid $uuid;

    #[ORM\Column(length: 255, nullable: true)]
    private ?string $denomination = null;

    #[ORM\Column(length: 9, nullable: true)]
    private ?string $siren = null;

    #[ORM\Column(type: 'string', enumType: TypeDeclarant::class, nullable: true)]
    private ?TypeDeclarant $type = null;

    #[ORM\Embedded(class: PersonneVO::class, columnPrefix: 'responsable_')]
    private ?PersonneVO $responsable = null;

    #[ORM\Embedded(class: AdresseVO::class, columnPrefix: 'adresse_')]
    private ?AdresseVO $adresse = null;

    public function __construct()
    {
        $this->uuid = AppUuid::generate();
    }

    public function getUuid(): AppUuid
    {
        return $this->uuid;
    }

    public function getDenomination(): ?string
    {
        return $this->denomination;
    }

    public function setDenomination(?string $denomination): self
    {
        $this->denomination = $denomination;
        return $this;
    }

    public function getSiren(): ?string
    {
        return $this->siren;
    }

    public function setSiren(?string $siren): self
    {
        $this->siren = $siren;
        return $this;
    }

    public function getType(): ?TypeDeclarant
    {
        return $this->type;
    }

    public function setType(?TypeDeclarant $type): self
    {
        $this->type = $type;
        return $this;
    }

    public function getResponsable(): ?PersonneVO
    {
        return $this->responsable;
    }

    public function setResponsable(?PersonneVO $responsable): self
    {
        $this->responsable = $responsable;
        return $this;
    }

    public function getAdresse(): ?AdresseVO
    {
        return $this->adresse;
    }

    public function setAdresse(?AdresseVO $adresse): self
    {
        $this->adresse = $adresse;
        return $this;
    }
}
```

### Traits Réutilisables

#### IdTrait

```php
<?php

declare(strict_types=1);

namespace App\Domain\Entity\Traits;

use Doctrine\ORM\Mapping as ORM;

trait IdTrait
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private ?int $id = null;

    public function getId(): ?int
    {
        return $this->id;
    }
}
```

#### TimestampableTrait

```php
<?php

declare(strict_types=1);

namespace App\Domain\Entity\Traits;

use Doctrine\ORM\Mapping as ORM;

trait TimestampableTrait
{
    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $createdAt;

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $updatedAt;

    #[ORM\PrePersist]
    public function setCreatedAtValue(): void
    {
        $this->createdAt = new \DateTimeImmutable();
        $this->updatedAt = new \DateTimeImmutable();
    }

    #[ORM\PreUpdate]
    public function setUpdatedAtValue(): void
    {
        $this->updatedAt = new \DateTimeImmutable();
    }

    public function getCreatedAt(): \DateTimeImmutable
    {
        return $this->createdAt;
    }

    public function getUpdatedAt(): \DateTimeImmutable
    {
        return $this->updatedAt;
    }
}
```

#### SoftDeletableTrait

```php
<?php

declare(strict_types=1);

namespace App\Domain\Entity\Traits;

use Doctrine\ORM\Mapping as ORM;

trait SoftDeletableTrait
{
    #[ORM\Column(type: 'datetime_immutable', nullable: true)]
    private ?\DateTimeImmutable $deletedAt = null;

    public function getDeletedAt(): ?\DateTimeImmutable
    {
        return $this->deletedAt;
    }

    public function setDeletedAt(?\DateTimeImmutable $deletedAt): self
    {
        $this->deletedAt = $deletedAt;
        return $this;
    }

    public function isDeleted(): bool
    {
        return $this->deletedAt !== null;
    }
}
```

### Entity avec Relations

```php
<?php

declare(strict_types=1);

namespace App\Domain\Entity;

use App\Domain\Entity\Traits\IdTrait;
use App\Domain\Entity\Traits\TimestampableTrait;
use App\Domain\ValueObject\AppUuid;
use App\Domain\Enum\Status;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\Table(name: 'declaration')]
class Declaration
{
    use IdTrait;
    use TimestampableTrait;

    #[ORM\Column(type: 'app_uuid', unique: true)]
    private readonly AppUuid $uuid;

    #[ORM\ManyToOne(targetEntity: Declarant::class)]
    #[ORM\JoinColumn(nullable: false)]
    private Declarant $declarant;

    #[ORM\Column(type: 'string', enumType: Status::class)]
    private Status $status = Status::DRAFT;

    /**
     * @var Collection<int, LigneDeclaration>
     */
    #[ORM\OneToMany(
        targetEntity: LigneDeclaration::class,
        mappedBy: 'declaration',
        cascade: ['persist', 'remove'],
        orphanRemoval: true
    )]
    private Collection $lignes;

    public function __construct(Declarant $declarant)
    {
        $this->uuid = AppUuid::generate();
        $this->declarant = $declarant;
        $this->lignes = new ArrayCollection();
    }

    public function getUuid(): AppUuid
    {
        return $this->uuid;
    }

    public function getDeclarant(): Declarant
    {
        return $this->declarant;
    }

    public function getStatus(): Status
    {
        return $this->status;
    }

    public function setStatus(Status $status): self
    {
        $this->status = $status;
        return $this;
    }

    /**
     * @return Collection<int, LigneDeclaration>
     */
    public function getLignes(): Collection
    {
        return $this->lignes;
    }

    public function addLigne(LigneDeclaration $ligne): self
    {
        if (!$this->lignes->contains($ligne)) {
            $this->lignes->add($ligne);
            $ligne->setDeclaration($this);
        }
        return $this;
    }

    public function removeLigne(LigneDeclaration $ligne): self
    {
        $this->lignes->removeElement($ligne);
        return $this;
    }
}
```

## ValueObjects (Embeddable)

### ValueObject Simple

```php
<?php

declare(strict_types=1);

namespace App\Domain\ValueObject;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Embeddable]
final readonly class AdresseVO implements \JsonSerializable
{
    public function __construct(
        #[ORM\Column(length: 255, nullable: true)]
        private ?string $voie = null,

        #[ORM\Column(length: 10, nullable: true)]
        private ?string $codePostal = null,

        #[ORM\Column(length: 100, nullable: true)]
        private ?string $ville = null,

        #[ORM\Column(length: 2, nullable: true)]
        private ?string $pays = null,
    ) {}

    public function getVoie(): ?string
    {
        return $this->voie;
    }

    public function getCodePostal(): ?string
    {
        return $this->codePostal;
    }

    public function getVille(): ?string
    {
        return $this->ville;
    }

    public function getPays(): ?string
    {
        return $this->pays;
    }

    public function getFullAddress(): string
    {
        return implode(', ', array_filter([
            $this->voie,
            $this->codePostal,
            $this->ville,
            $this->pays,
        ]));
    }

    public function isEmpty(): bool
    {
        return $this->voie === null
            && $this->codePostal === null
            && $this->ville === null
            && $this->pays === null;
    }

    public function jsonSerialize(): array
    {
        return [
            'voie' => $this->voie,
            'codePostal' => $this->codePostal,
            'ville' => $this->ville,
            'pays' => $this->pays,
        ];
    }
}
```

### ValueObject Personne

```php
<?php

declare(strict_types=1);

namespace App\Domain\ValueObject;

use App\Domain\Enum\Civilite;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Embeddable]
final readonly class PersonneVO implements \JsonSerializable
{
    public function __construct(
        #[ORM\Column(type: 'string', enumType: Civilite::class, nullable: true)]
        private ?Civilite $civilite = null,

        #[ORM\Column(length: 100, nullable: true)]
        private ?string $nom = null,

        #[ORM\Column(length: 100, nullable: true)]
        private ?string $prenom = null,

        #[ORM\Column(length: 255, nullable: true)]
        private ?string $email = null,

        #[ORM\Column(length: 20, nullable: true)]
        private ?string $telephone = null,
    ) {}

    public function getCivilite(): ?Civilite
    {
        return $this->civilite;
    }

    public function getNom(): ?string
    {
        return $this->nom;
    }

    public function getPrenom(): ?string
    {
        return $this->prenom;
    }

    public function getEmail(): ?string
    {
        return $this->email;
    }

    public function getTelephone(): ?string
    {
        return $this->telephone;
    }

    public function getFullName(): string
    {
        return trim(sprintf('%s %s', $this->prenom ?? '', $this->nom ?? ''));
    }

    public function jsonSerialize(): array
    {
        return [
            'civilite' => $this->civilite?->value,
            'nom' => $this->nom,
            'prenom' => $this->prenom,
            'email' => $this->email,
            'telephone' => $this->telephone,
        ];
    }
}
```

### ValueObject UUID

```php
<?php

declare(strict_types=1);

namespace App\Domain\ValueObject;

use Symfony\Component\Uid\Uuid;

final readonly class AppUuid implements \Stringable, \JsonSerializable
{
    private function __construct(
        private string $value,
    ) {}

    public static function generate(): self
    {
        return new self(Uuid::v7()->toRfc4122());
    }

    public static function fromString(string $value): self
    {
        if (!Uuid::isValid($value)) {
            throw new \InvalidArgumentException(sprintf('Invalid UUID: %s', $value));
        }

        return new self($value);
    }

    public function getValue(): string
    {
        return $this->value;
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }

    public function __toString(): string
    {
        return $this->value;
    }

    public function jsonSerialize(): string
    {
        return $this->value;
    }
}
```

### ValueObject Money

```php
<?php

declare(strict_types=1);

namespace App\Domain\ValueObject;

final readonly class MoneyVO implements \JsonSerializable
{
    public function __construct(
        private int $amount,       // En centimes
        private string $currency = 'EUR',
    ) {}

    public static function fromFloat(float $amount, string $currency = 'EUR'): self
    {
        return new self((int) round($amount * 100), $currency);
    }

    public function getAmount(): int
    {
        return $this->amount;
    }

    public function getAmountAsFloat(): float
    {
        return $this->amount / 100;
    }

    public function getCurrency(): string
    {
        return $this->currency;
    }

    public function add(self $other): self
    {
        $this->assertSameCurrency($other);
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function subtract(self $other): self
    {
        $this->assertSameCurrency($other);
        return new self($this->amount - $other->amount, $this->currency);
    }

    public function multiply(float $factor): self
    {
        return new self((int) round($this->amount * $factor), $this->currency);
    }

    public function equals(self $other): bool
    {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }

    public function format(): string
    {
        return number_format($this->getAmountAsFloat(), 2, ',', ' ') . ' ' . $this->currency;
    }

    private function assertSameCurrency(self $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException(
                sprintf('Cannot operate on different currencies: %s and %s', $this->currency, $other->currency)
            );
        }
    }

    public function jsonSerialize(): array
    {
        return [
            'amount' => $this->amount,
            'currency' => $this->currency,
        ];
    }
}
```

## Enums

### Enum Simple

```php
<?php

declare(strict_types=1);

namespace App\Domain\Enum;

enum Status: string
{
    case DRAFT = 'draft';
    case PENDING = 'pending';
    case VALIDATED = 'validated';
    case REJECTED = 'rejected';
    case CANCELLED = 'cancelled';

    public function getLabel(): string
    {
        return match ($this) {
            self::DRAFT => 'Brouillon',
            self::PENDING => 'En attente',
            self::VALIDATED => 'Validé',
            self::REJECTED => 'Rejeté',
            self::CANCELLED => 'Annulé',
        };
    }

    public function getColor(): string
    {
        return match ($this) {
            self::DRAFT => 'gray',
            self::PENDING => 'yellow',
            self::VALIDATED => 'green',
            self::REJECTED => 'red',
            self::CANCELLED => 'gray',
        };
    }

    public function canTransitionTo(self $target): bool
    {
        return match ($this) {
            self::DRAFT => in_array($target, [self::PENDING, self::CANCELLED], true),
            self::PENDING => in_array($target, [self::VALIDATED, self::REJECTED], true),
            self::VALIDATED => false,
            self::REJECTED => in_array($target, [self::DRAFT], true),
            self::CANCELLED => false,
        };
    }
}
```

### Enum avec Méthodes Complexes

```php
<?php

declare(strict_types=1);

namespace App\Domain\Enum;

enum Formulaire: string
{
    case TVA_3310CA3 = '3310CA3';
    case TVA_3519 = '3519';
    case TVA_3517S = '3517S';
    case IS_2065 = '2065';

    public function getLabel(): string
    {
        return match ($this) {
            self::TVA_3310CA3 => 'Déclaration de TVA CA3',
            self::TVA_3519 => 'Déclaration de TVA simplifiée',
            self::TVA_3517S => 'Relevé de TVA S',
            self::IS_2065 => 'Déclaration d\'impôt sur les sociétés',
        };
    }

    public function getCategory(): string
    {
        return match ($this) {
            self::TVA_3310CA3, self::TVA_3519, self::TVA_3517S => 'TVA',
            self::IS_2065 => 'IS',
        };
    }

    public function getPeriodicite(): string
    {
        return match ($this) {
            self::TVA_3310CA3 => 'mensuelle',
            self::TVA_3519, self::TVA_3517S => 'trimestrielle',
            self::IS_2065 => 'annuelle',
        };
    }

    /**
     * @return array<self>
     */
    public static function getTVAFormulaires(): array
    {
        return [
            self::TVA_3310CA3,
            self::TVA_3519,
            self::TVA_3517S,
        ];
    }
}
```

### Enum Civilité

```php
<?php

declare(strict_types=1);

namespace App\Domain\Enum;

enum Civilite: string
{
    case MONSIEUR = 'M';
    case MADAME = 'Mme';

    public function getLabel(): string
    {
        return match ($this) {
            self::MONSIEUR => 'Monsieur',
            self::MADAME => 'Madame',
        };
    }
}
```

## Domain Events

### AbstractEvent

```php
<?php

declare(strict_types=1);

namespace App\Domain\Event;

abstract class AbstractEvent
{
    private readonly \DateTimeImmutable $occurredAt;
    private readonly array $payload;

    public function __construct(array $payload = [])
    {
        $this->occurredAt = new \DateTimeImmutable();
        $this->payload = $payload;
    }

    abstract public function getEvent(): string;

    public function getOccurredAt(): \DateTimeImmutable
    {
        return $this->occurredAt;
    }

    public function getPayload(): array
    {
        return $this->payload;
    }
}
```

### Event Concret

```php
<?php

declare(strict_types=1);

namespace App\Domain\Event;

use App\Domain\ValueObject\AppUuid;

final class DeclarantCreatedEvent extends AbstractEvent
{
    public function __construct(AppUuid $uuid)
    {
        parent::__construct([
            'id' => (string) $uuid,
        ]);
    }

    public function getEvent(): string
    {
        return 'declarant.created';
    }

    public function getDeclarantId(): string
    {
        return $this->getPayload()['id'];
    }
}
```

### Event Listener

```php
<?php

declare(strict_types=1);

namespace App\Application\Declarant\EventListener;

use App\Domain\Event\DeclarantCreatedEvent;
use App\Domain\Service\NotificationServiceInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final readonly class DeclarantCreatedEventListener
{
    public function __construct(
        private NotificationServiceInterface $notificationService,
    ) {}

    public function __invoke(DeclarantCreatedEvent $event): void
    {
        $this->notificationService->notifyDeclarantCreated(
            $event->getDeclarantId()
        );
    }
}
```

## Exceptions

### Exception de Base

```php
<?php

declare(strict_types=1);

namespace App\Domain\Exception;

abstract class DomainException extends \Exception
{
    public function __construct(
        string $message,
        int $code = 0,
        ?\Throwable $previous = null,
    ) {
        parent::__construct($message, $code, $previous);
    }
}
```

### NotFound Exception

```php
<?php

declare(strict_types=1);

namespace App\Domain\Exception\NotFound;

use App\Domain\Exception\DomainException;

class ResourceNotFoundException extends DomainException
{
    public function __construct(string $resource, mixed $id)
    {
        parent::__construct(
            sprintf('%s with id "%s" not found', $resource, $id)
        );
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Domain\Exception\NotFound;

final class DeclarantNotFoundException extends ResourceNotFoundException
{
    public function __construct(mixed $id)
    {
        parent::__construct('Declarant', $id);
    }
}
```

### Validation Exception

```php
<?php

declare(strict_types=1);

namespace App\Domain\Exception\Validator;

use App\Domain\Exception\DomainException;

final class ValidationException extends DomainException
{
    /**
     * @param array<string, array<string>> $errors
     */
    public function __construct(
        private readonly array $errors,
    ) {
        parent::__construct('Validation failed');
    }

    /**
     * @return array<string, array<string>>
     */
    public function getErrors(): array
    {
        return $this->errors;
    }
}
```

## Checklist Entités

- [ ] `declare(strict_types=1)` en haut du fichier
- [ ] Entité avec traits si applicable (IdTrait, TimestampableTrait)
- [ ] UUID généré dans le constructeur
- [ ] Propriétés typées avec visibilité explicite
- [ ] Getters/Setters avec return type
- [ ] Relations avec cascade approprié
- [ ] ValueObjects en `readonly class` avec `#[Embeddable]`
- [ ] Enums avec méthodes métier
- [ ] Events héritent de AbstractEvent
- [ ] Exceptions organisées par type HTTP