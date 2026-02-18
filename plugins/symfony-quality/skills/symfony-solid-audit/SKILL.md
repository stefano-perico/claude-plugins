---
name: symfony-solid-audit
description: Audit de code Symfony pour vérifier le respect des principes SOLID. Utiliser ce skill pour analyser une classe, un répertoire, ou une feature complète et identifier les violations SRP, OCP, LSP, ISP et DIP. Déclencher quand l'utilisateur demande un audit SOLID, une revue d'architecture, ou veut vérifier la qualité de son code Symfony.
---

# Audit SOLID Symfony

Analyser le code PHP/Symfony pour détecter les violations des 5 principes SOLID et générer un rapport Markdown avec score et recommandations.

## Workflow

1. **Identifier la cible** : classe spécifique, répertoire, ou feature
2. **Scanner les fichiers** : utiliser Glob pour lister les fichiers PHP
3. **Analyser chaque principe** : appliquer les règles de `references/solid-rules.md`
4. **Calculer le score** : HIGH=-10, MEDIUM=-5, LOW=-2 (base 100)
5. **Générer le rapport** : format Markdown structuré

## Analyse par principe

### S - Single Responsibility

Vérifier :
- Lignes par classe (> 300 = violation)
- Méthodes publiques (> 7 = warning)
- Dépendances injectées (> 5 = warning)
- Controller avec logique métier

### O - Open/Closed

Chercher :
- `switch` sur types/statuts
- `instanceof` répétés
- Chaînes `if/elseif` sur type

Recommander : Tagged Services, Strategy pattern

### L - Liskov Substitution

Détecter :
- `throw new \Exception` dans override
- Méthodes qui retournent null si parent retourne objet
- Préconditions renforcées dans enfant

### I - Interface Segregation

Identifier :
- Interfaces > 5 méthodes
- Implémentations vides `{ }`
- `throw new \Exception('Not implemented')`

### D - Dependency Inversion

Repérer :
- `new ServiceClass()` dans constructeur
- Types concrets au lieu d'interfaces
- Appels statiques `Service::method()`
- Dans Domain : imports de Doctrine ou HttpFoundation

## Format du rapport

```markdown
# Audit SOLID - [Cible]

**Date** : YYYY-MM-DD
**Score** : XX/100 (Évaluation)

## Résumé

| Principe | Violations | Sévérité max |
|----------|------------|--------------|
| SRP | X | HIGH/MEDIUM/LOW |
| OCP | X | ... |
| LSP | X | ... |
| ISP | X | ... |
| DIP | X | ... |

## Violations détaillées

### [S] Single Responsibility

#### `App\Service\UserService` - HIGH
- **Problème** : 15 méthodes publiques, 8 dépendances
- **Recommandation** : Extraire UserAuthService, UserNotificationService

### [D] Dependency Inversion

#### `App\Domain\Order\OrderProcessor` - HIGH
- **Problème** : `use Doctrine\ORM\EntityManagerInterface`
- **Recommandation** : Créer OrderRepositoryInterface dans Domain

## Actions prioritaires

1. [ ] Refactorer UserService (impact: -30 points)
2. [ ] Créer interfaces pour repositories Domain
3. [ ] Remplacer switch par Tagged Services
```

## Commandes utiles

```bash
# Lister classes > 300 lignes
find src -name "*.php" -exec wc -l {} \; | awk '$1 > 300'

# Trouver switch statements
grep -rn "switch\s*(" src/

# Détecter new dans services
grep -rn "= new " src/Service/
```

## Référence

Consulter `references/solid-rules.md` pour les règles détaillées, patterns corrects, et exemples de code.
