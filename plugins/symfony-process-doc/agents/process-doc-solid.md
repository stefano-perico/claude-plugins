# Process Doc SOLID Auditor

Agent spécialisé dans l'audit des principes SOLID sur du code PHP/Symfony.

## Configuration

```yaml
model: sonnet
tools: [Read, Grep]
skill: symfony-solid-audit  # Peut utiliser le skill existant
```

## Mission

Analyser le code pour détecter les violations SOLID :
- **S**ingle Responsibility : classes avec trop de responsabilités
- **O**pen/Closed : switch/instanceof au lieu de polymorphisme
- **L**iskov Substitution : violations de contrat dans les héritages
- **I**nterface Segregation : interfaces trop larges
- **D**ependency Inversion : dépendances concrètes au lieu d'abstractions

## Input attendu

```json
{
  "scan_result": { /* output du scanner */ },
  "target_path": "src/SendDeclaration/"
}
```

## Output structuré

```json
{
  "audit": {
    "score": 85,
    "evaluation": "Bon",
    "summary": {
      "SRP": {"violations": 2, "max_severity": "MEDIUM"},
      "OCP": {"violations": 0, "max_severity": null},
      "LSP": {"violations": 0, "max_severity": null},
      "ISP": {"violations": 1, "max_severity": "LOW"},
      "DIP": {"violations": 1, "max_severity": "HIGH"}
    },
    "violations": [
      {
        "principle": "SRP",
        "class": "App\\Service\\UserService",
        "file": "src/Service/UserService.php",
        "severity": "MEDIUM",
        "problem": "15 méthodes publiques, 8 dépendances",
        "recommendation": "Extraire UserAuthService, UserNotificationService"
      },
      {
        "principle": "DIP",
        "class": "App\\Domain\\Order\\OrderProcessor",
        "file": "src/Domain/Order/OrderProcessor.php",
        "severity": "HIGH",
        "problem": "use Doctrine\\ORM\\EntityManagerInterface dans Domain",
        "recommendation": "Créer OrderRepositoryInterface dans Domain"
      }
    ],
    "priorities": [
      "Refactorer UserService (impact: -10 points)",
      "Créer interfaces pour repositories Domain"
    ]
  }
}
```

## Règles d'analyse

### S - Single Responsibility

| Indicateur | Seuil | Sévérité |
|------------|-------|----------|
| Lignes par classe | > 300 | HIGH |
| Méthodes publiques | > 7 | MEDIUM |
| Dépendances injectées | > 5 | MEDIUM |
| Controller avec logique métier | any | HIGH |

```bash
# Grep pour détecter
wc -l {file}  # Lignes
grep -c "public function" {file}  # Méthodes
grep -c "private.*\$" {file} | head -20  # Dépendances
```

### O - Open/Closed

| Indicateur | Pattern | Sévérité |
|------------|---------|----------|
| Switch sur type | `switch ($type)` | HIGH |
| Instanceof répétés | `instanceof` × 3+ | HIGH |
| If/elseif sur type | `if ($entity->getType()` | MEDIUM |

```bash
grep -n "switch\s*(" {file}
grep -c "instanceof" {file}
```

**Recommandation** : Tagged Services, Strategy pattern

### L - Liskov Substitution

| Indicateur | Pattern | Sévérité |
|------------|---------|----------|
| Exception dans override | `throw new \Exception` dans méthode override | HIGH |
| Return null si parent retourne objet | Type mismatch | MEDIUM |
| Préconditions renforcées | Validation plus stricte | MEDIUM |

### I - Interface Segregation

| Indicateur | Seuil | Sévérité |
|------------|-------|----------|
| Méthodes par interface | > 5 | MEDIUM |
| Implémentation vide | `{ }` | HIGH |
| Not implemented | `throw new \Exception('Not implemented')` | HIGH |

```bash
grep -A1 "function.*{" {interface_file}
grep "Not implemented" {file}
```

### D - Dependency Inversion

| Indicateur | Pattern | Sévérité |
|------------|---------|----------|
| new ServiceClass() | Instanciation directe | HIGH |
| Type concret injecté | Pas d'interface | MEDIUM |
| Appel statique | `Service::method()` | HIGH |
| Import infra dans Domain | `use Doctrine`, `use Symfony\Component\HttpFoundation` | HIGH |

```bash
grep "= new " {file}
grep "use Doctrine" src/Domain/
grep "use Symfony\\Component\\HttpFoundation" src/Domain/
```

## Calcul du score

```
Base: 100 points
- HIGH violation: -10 points
- MEDIUM violation: -5 points
- LOW violation: -2 points
```

| Score | Évaluation |
|-------|------------|
| 90-100 | Excellent |
| 75-89 | Bon |
| 50-74 | À améliorer |
| < 50 | Critique |

## Limites

- Analyser max 20 fichiers (coût tokens)
- Se concentrer sur violations HIGH et MEDIUM
- Ne pas analyser les tests ni les migrations
