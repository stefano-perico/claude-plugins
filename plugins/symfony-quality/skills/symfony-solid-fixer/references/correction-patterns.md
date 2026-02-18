# Patterns de Correction SOLID pour Symfony

## [D] Dependency Inversion - Patterns de correction

### Pattern 1 : Extraction d'interface Repository

**Avant** (violation) :
```php
// src/Application/Order/Manager/OrderManager.php
class OrderManager
{
    public function __construct(
        private OrderRepository $orderRepository  // Type concret
    ) {}
}
```

**Après** (correction) :
```php
// src/Domain/Repository/OrderRepositoryInterface.php
interface OrderRepositoryInterface extends RepositoryInterface
{
    public function ofId(mixed $id): Order;
    public function save(Order $order): void;
}

// src/Infrastructure/Persistence/Doctrine/OrderRepository.php
class OrderRepository extends DoctrineRepository implements OrderRepositoryInterface
{
    public function ofId(mixed $id): Order
    {
        return $this->find($id) ?? throw new OrderNotFoundException($id);
    }

    public function save(Order $order): void
    {
        $this->getEntityManager()->persist($order);
        $this->getEntityManager()->flush();
    }
}

// src/Application/Order/Manager/OrderManager.php
class OrderManager
{
    public function __construct(
        private OrderRepositoryInterface $orderRepository  // Interface
    ) {}
}
```

### Pattern 2 : Remplacement de `new` par injection

**Avant** :
```php
class InvoiceGenerator
{
    public function generate(Order $order): Invoice
    {
        $calculator = new TaxCalculator();  // Violation DIP
        $tax = $calculator->calculate($order);
        return new Invoice($order, $tax);
    }
}
```

**Après** :
```php
// src/Domain/Service/TaxCalculatorInterface.php
interface TaxCalculatorInterface
{
    public function calculate(Order $order): Money;
}

// src/Application/Invoice/Service/InvoiceGenerator.php
class InvoiceGenerator
{
    public function __construct(
        private TaxCalculatorInterface $taxCalculator
    ) {}

    public function generate(Order $order): Invoice
    {
        $tax = $this->taxCalculator->calculate($order);
        return new Invoice($order, $tax);
    }
}
```

### Pattern 3 : Isolation du Domain

**Avant** (imports infrastructure dans Domain) :
```php
// src/Domain/Entity/User.php
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\HttpFoundation\File\UploadedFile;  // VIOLATION

#[ORM\Entity]
class User
{
    public function setAvatar(UploadedFile $file): void { ... }
}
```

**Après** :
```php
// src/Domain/Entity/User.php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class User
{
    public function setAvatarPath(string $path): void { ... }
}

// src/Application/User/Manager/UserAvatarManager.php
class UserAvatarManager
{
    public function __construct(
        private FileUploaderInterface $uploader,
        private UserRepositoryInterface $userRepository
    ) {}

    public function uploadAvatar(User $user, UploadedFile $file): void
    {
        $path = $this->uploader->upload($file);
        $user->setAvatarPath($path);
        $this->userRepository->save($user);
    }
}
```

---

## [S] Single Responsibility - Patterns de correction

### Pattern 1 : Extraction de Service

**Avant** (God Service) :
```php
class UserService
{
    public function create(array $data): User { ... }
    public function update(User $user, array $data): void { ... }
    public function delete(User $user): void { ... }
    public function sendWelcomeEmail(User $user): void { ... }
    public function sendPasswordResetEmail(User $user): void { ... }
    public function generateReport(): array { ... }
    public function exportToCsv(): string { ... }
}
```

**Après** (séparation par responsabilité) :
```php
// Gestion CRUD
class UserManager
{
    public function create(CreateUserCommand $cmd): User { ... }
    public function update(UpdateUserCommand $cmd): void { ... }
    public function delete(DeleteUserCommand $cmd): void { ... }
}

// Notifications
class UserNotificationService
{
    public function sendWelcomeEmail(User $user): void { ... }
    public function sendPasswordResetEmail(User $user): void { ... }
}

// Reporting
class UserReportService
{
    public function generateReport(): array { ... }
    public function exportToCsv(): string { ... }
}
```

### Pattern 2 : Extraction CQRS

**Avant** (Controller avec logique) :
```php
class UserController
{
    public function create(Request $request): Response
    {
        $data = json_decode($request->getContent(), true);
        // Validation...
        $user = new User();
        $user->setEmail($data['email']);
        // Hash password...
        // Send email...
        $this->em->persist($user);
        $this->em->flush();
        return new JsonResponse(['id' => $user->getId()]);
    }
}
```

**Après** :
```php
// src/Application/User/Command/CreateUserCommand.php
final readonly class CreateUserCommand implements InternalCommandInterface
{
    public function __construct(
        public string $email,
        public string $password,
    ) {}
}

// src/Application/User/Command/CreateUserCommandHandler.php
#[AsMessageHandler]
final readonly class CreateUserCommandHandler
{
    public function __construct(private UserManager $manager) {}

    public function __invoke(CreateUserCommand $cmd): User
    {
        return $this->manager->create($cmd);
    }
}

// src/Infrastructure/Controller/UserController.php
class UserController
{
    public function create(Request $request, MessageBusInterface $bus): Response
    {
        $cmd = new CreateUserCommand(
            email: $request->get('email'),
            password: $request->get('password'),
        );
        $user = $bus->dispatch($cmd);
        return new JsonResponse(['id' => $user->getId()]);
    }
}
```

---

## [O] Open/Closed - Patterns de correction

### Pattern 1 : Switch vers Tagged Services

**Avant** :
```php
class NotificationService
{
    public function send(Notification $notification): void
    {
        switch ($notification->getChannel()) {
            case 'email':
                $this->sendEmail($notification);
                break;
            case 'sms':
                $this->sendSms($notification);
                break;
            case 'push':
                $this->sendPush($notification);
                break;
            default:
                throw new \InvalidArgumentException('Unknown channel');
        }
    }
}
```

**Après** :
```php
// src/Domain/Service/NotifierInterface.php
#[AutoconfigureTag('app.notifier')]
interface NotifierInterface
{
    public function supports(Notification $notification): bool;
    public function send(Notification $notification): void;
}

// src/Application/Notification/Service/EmailNotifier.php
class EmailNotifier implements NotifierInterface
{
    public function supports(Notification $n): bool
    {
        return $n->getChannel() === 'email';
    }

    public function send(Notification $n): void { ... }
}

// src/Application/Notification/Service/SmsNotifier.php
class SmsNotifier implements NotifierInterface { ... }

// src/Application/Notification/Service/PushNotifier.php
class PushNotifier implements NotifierInterface { ... }

// src/Application/Notification/Service/NotificationService.php
class NotificationService
{
    public function __construct(
        #[TaggedIterator('app.notifier')]
        private iterable $notifiers
    ) {}

    public function send(Notification $notification): void
    {
        foreach ($this->notifiers as $notifier) {
            if ($notifier->supports($notification)) {
                $notifier->send($notification);
                return;
            }
        }
        throw new NoNotifierFoundException($notification->getChannel());
    }
}
```

### Pattern 2 : instanceof vers Polymorphisme

**Avant** :
```php
class DocumentProcessor
{
    public function process(Document $doc): string
    {
        if ($doc instanceof PdfDocument) {
            return $this->processPdf($doc);
        }
        if ($doc instanceof WordDocument) {
            return $this->processWord($doc);
        }
        throw new \InvalidArgumentException();
    }
}
```

**Après** :
```php
// src/Domain/Entity/Document.php
abstract class Document
{
    abstract public function getContent(): string;
}

// src/Domain/Entity/PdfDocument.php
class PdfDocument extends Document
{
    public function getContent(): string
    {
        // PDF-specific extraction
    }
}

// src/Domain/Entity/WordDocument.php
class WordDocument extends Document
{
    public function getContent(): string
    {
        // Word-specific extraction
    }
}

// src/Application/Document/Service/DocumentProcessor.php
class DocumentProcessor
{
    public function process(Document $doc): string
    {
        return $doc->getContent();  // Polymorphisme
    }
}
```

---

## [I] Interface Segregation - Patterns de correction

### Pattern 1 : Découpage d'interface fat

**Avant** :
```php
interface UserRepositoryInterface
{
    public function find(int $id): ?User;
    public function findAll(): array;
    public function save(User $user): void;
    public function delete(User $user): void;
    public function findByEmail(string $email): ?User;
    public function findActiveUsers(): array;
    public function countByRole(string $role): int;
    public function getStatistics(): array;
}
```

**Après** :
```php
// Lecture basique
interface UserReaderInterface
{
    public function find(int $id): ?User;
    public function findAll(): array;
}

// Écriture
interface UserWriterInterface
{
    public function save(User $user): void;
    public function delete(User $user): void;
}

// Requêtes spécifiques
interface UserQueryInterface
{
    public function findByEmail(string $email): ?User;
    public function findActiveUsers(): array;
}

// Statistiques (optionnel)
interface UserStatisticsInterface
{
    public function countByRole(string $role): int;
    public function getStatistics(): array;
}

// L'implémentation peut implémenter toutes ou certaines
class UserRepository implements
    UserReaderInterface,
    UserWriterInterface,
    UserQueryInterface,
    UserStatisticsInterface
{ ... }
```

---

## [L] Liskov Substitution - Patterns de correction

### Pattern 1 : Correction de préconditions

**Avant** (violation - préconditions renforcées) :
```php
interface PaymentProcessorInterface
{
    public function process(Payment $payment): void;
}

class CreditCardProcessor implements PaymentProcessorInterface
{
    public function process(Payment $payment): void
    {
        if ($payment->getAmount() < 10) {  // VIOLATION: nouvelle précondition
            throw new MinimumAmountException();
        }
        // Process...
    }
}
```

**Après** :
```php
interface PaymentProcessorInterface
{
    public function supports(Payment $payment): bool;
    public function process(Payment $payment): void;
}

class CreditCardProcessor implements PaymentProcessorInterface
{
    public function supports(Payment $payment): bool
    {
        return $payment->getAmount() >= 10;  // Validation via supports()
    }

    public function process(Payment $payment): void
    {
        // Process without additional validation
    }
}
```

### Pattern 2 : Correction de return type

**Avant** (violation - null au lieu d'objet) :
```php
interface CacheInterface
{
    public function get(string $key): mixed;
}

class RedisCache implements CacheInterface
{
    public function get(string $key): mixed
    {
        $value = $this->redis->get($key);
        return $value ?: null;  // Parent retourne toujours une valeur
    }
}
```

**Après** :
```php
interface CacheInterface
{
    public function get(string $key): mixed;
    public function has(string $key): bool;
}

class RedisCache implements CacheInterface
{
    public function get(string $key): mixed
    {
        if (!$this->has($key)) {
            throw new CacheKeyNotFoundException($key);
        }
        return $this->redis->get($key);
    }

    public function has(string $key): bool
    {
        return $this->redis->exists($key);
    }
}
```

---

## Configuration services.yaml

### Autowiring d'interfaces

```yaml
# config/services.yaml
services:
    # Bind interface to implementation
    App\Domain\Repository\UserRepositoryInterface:
        alias: App\Infrastructure\Persistence\Doctrine\UserRepository

    # Tagged services auto-discovery
    _instanceof:
        App\Domain\Service\NotifierInterface:
            tags: ['app.notifier']
```

### Service locator pour strategies

```yaml
services:
    App\Application\Payment\Service\PaymentService:
        arguments:
            $processors: !tagged_iterator app.payment_processor
```
