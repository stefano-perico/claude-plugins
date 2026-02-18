# Règles SOLID pour Symfony

## S - Single Responsibility Principle (SRP)

### Violations typiques

| Pattern | Exemple | Sévérité |
|---------|---------|----------|
| Controller > 200 lignes | Logique métier dans controller | HIGH |
| Service avec > 5 dépendances | God service | HIGH |
| Méthode > 30 lignes | Faire trop de choses | MEDIUM |
| Repository avec logique métier | Mélange couches | HIGH |
| Entity avec méthodes métier complexes | Anemic domain inversé | MEDIUM |

### Indicateurs

```
# Métriques à vérifier
- Nombre de méthodes publiques par classe (> 7 = warning)
- Nombre de dépendances injectées (> 5 = warning)
- Lignes par méthode (> 30 = warning)
- Lignes par classe (> 300 = warning)
```

### Corrections Symfony

- Extraire en Service dédié
- Utiliser Command/Query Handlers
- Créer des Value Objects
- Utiliser des Event Listeners

---

## O - Open/Closed Principle (OCP)

### Violations typiques

| Pattern | Exemple | Sévérité |
|---------|---------|----------|
| Switch/case sur type | `switch($type)` pour comportement | HIGH |
| instanceof répétés | Vérifications de type multiples | HIGH |
| Modification constante d'une classe | Ajout de if/else | MEDIUM |

### Patterns Symfony corrects

```php
// ❌ Violation OCP
class NotificationService {
    public function send(Notification $n): void {
        switch($n->getType()) {
            case 'email': // ...
            case 'sms': // ...
            case 'push': // ... <- Modification requise pour nouveau type
        }
    }
}

// ✅ Respect OCP avec Tagged Services
#[AutoconfigureTag('app.notifier')]
interface NotifierInterface {
    public function supports(Notification $n): bool;
    public function send(Notification $n): void;
}

class NotificationService {
    public function __construct(
        #[TaggedIterator('app.notifier')]
        private iterable $notifiers
    ) {}
}
```

---

## L - Liskov Substitution Principle (LSP)

### Violations typiques

| Pattern | Exemple | Sévérité |
|---------|---------|----------|
| Override qui lance exception | `throw new \Exception('Not supported')` | HIGH |
| Override qui change le comportement | Préconditions renforcées | HIGH |
| Return type différent logiquement | Null au lieu d'objet attendu | MEDIUM |

### Vérifications

```
# Chercher dans les overrides:
- throw new \Exception
- throw new \RuntimeException
- return null (si parent retourne objet)
- Conditions plus restrictives que parent
```

---

## I - Interface Segregation Principle (ISP)

### Violations typiques

| Pattern | Exemple | Sévérité |
|---------|---------|----------|
| Interface > 5 méthodes | Interface "fat" | MEDIUM |
| Implémentation vide | `public function x(): void {}` | HIGH |
| Implémentation qui lance exception | `throw new \Exception('Not implemented')` | HIGH |

### Patterns Symfony corrects

```php
// ❌ Violation ISP
interface UserManagerInterface {
    public function create(User $u): void;
    public function update(User $u): void;
    public function delete(User $u): void;
    public function sendEmail(User $u): void;    // Pas lié
    public function generateReport(): string;     // Pas lié
}

// ✅ Respect ISP
interface UserWriterInterface {
    public function create(User $u): void;
    public function update(User $u): void;
    public function delete(User $u): void;
}

interface UserNotifierInterface {
    public function sendEmail(User $u): void;
}
```

---

## D - Dependency Inversion Principle (DIP)

### Violations typiques

| Pattern | Exemple | Sévérité |
|---------|---------|----------|
| `new` dans constructeur | Couplage fort | HIGH |
| Type concret en paramètre | `function x(MySqlRepo $r)` | HIGH |
| Dépendance infrastructure dans Domain | Entity qui utilise Service | HIGH |
| Static call | `MyService::doSomething()` | HIGH |

### Patterns Symfony corrects

```php
// ❌ Violation DIP
class OrderService {
    private MySqlOrderRepository $repo;  // Type concret

    public function __construct() {
        $this->repo = new MySqlOrderRepository();  // new dans constructeur
    }
}

// ✅ Respect DIP
class OrderService {
    public function __construct(
        private OrderRepositoryInterface $repo  // Interface
    ) {}
}
```

### Vérifications spécifiques Symfony

```
# Dans src/Domain ou src/Application:
- Aucun use Doctrine\*
- Aucun use Symfony\Component\HttpFoundation\*
- Aucun use App\Infrastructure\*

# Dans tous les services:
- Constructeur utilise interfaces (sauf Value Objects)
- Pas de new pour les services
- Pas d'appels statiques (sauf helpers purs)
```

---

## Commandes grep utiles

```bash
# SRP: Classes trop grandes
find src -name "*.php" -exec wc -l {} \; | awk '$1 > 300'

# OCP: Switch statements
grep -rn "switch\s*(" src/

# LSP: Exceptions dans overrides
grep -rn "throw new.*Exception" src/ | grep -v "Controller"

# ISP: Méthodes vides
grep -rn "{\s*}" src/ --include="*.php"

# DIP: new dans constructeurs
grep -rn "= new " src/ | grep -v "Value\|DTO\|Entity"

# DIP: Appels statiques
grep -rn "::" src/ --include="*.php" | grep -v "self::\|static::\|parent::"
```

---

## Scoring

| Sévérité | Points |
|----------|--------|
| HIGH | -10 |
| MEDIUM | -5 |
| LOW | -2 |

**Score final = 100 + somme des points**

| Score | Évaluation |
|-------|------------|
| 90-100 | Excellent |
| 70-89 | Bon |
| 50-69 | À améliorer |
| < 50 | Critique |
