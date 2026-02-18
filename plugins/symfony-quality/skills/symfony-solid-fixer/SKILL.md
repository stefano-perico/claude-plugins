---
name: symfony-solid-fixer
description: |
  Orchestration des corrections de violations SOLID dans du code Symfony.
  Utiliser ce skill après un audit SOLID (symfony-solid-audit ou
  symfony-process-doc --audit) pour appliquer les corrections.
  Le rapport de symfony-process-doc --audit est préféré car il fournit
  le contexte architectural complet (diagrammes, structure, dépendances).
  Déclencher quand l'utilisateur demande de corriger les violations SOLID,
  appliquer les recommandations d'audit, ou refactorer du code pour respecter
  les principes SOLID. Ce skill utilise des subagents spécialisés pour
  chaque type de correction.
---

# Symfony SOLID Fixer

Orchestrer les corrections des violations SOLID identifiées par un audit, en utilisant des subagents spécialisés pour chaque type de refactoring.

## Sources d'entrée

Le skill accepte deux types de rapports :

### Option 1 : `symfony-process-doc --audit` (recommandé)

Rapport complet avec contexte architectural :
- Section 1 : Diagrammes de flux (flowchart, stateDiagram)
- Section 4 : Architecture (structure fichiers, ports & adapters)
- Section 7 : Points d'attention existants
- **Section 8 : Audit SOLID** (violations à corriger)

### Option 2 : `symfony-solid-audit`

Rapport minimal avec uniquement les violations SOLID. Format attendu :

```markdown
## Violations détaillées

### [S] Single Responsibility
#### `App\Service\UserService` - HIGH
- **Problème** : 15 méthodes publiques, 8 dépendances
- **Recommandation** : Extraire UserAuthService, UserNotificationService

### [D] Dependency Inversion
#### `App\Domain\Order\OrderProcessor` - HIGH
- **Problème** : `use Doctrine\ORM\EntityManagerInterface`
- **Recommandation** : Créer OrderRepositoryInterface dans Domain
```

### Détection automatique

| Indicateur | Type de rapport |
|------------|-----------------|
| Contient "## 1. Vue d'ensemble" | `symfony-process-doc` |
| Contient "# Audit SOLID -" | `symfony-solid-audit` |

## Workflow principal

```
┌─────────────────────────────────────────────────────────────────┐
│                    SYMFONY SOLID FIXER                          │
├─────────────────────────────────────────────────────────────────┤
│  1. PARSING         Parser le rapport d'audit                   │
│         ↓                                                       │
│  2. PRIORISATION    Trier par sévérité (HIGH → MEDIUM → LOW)   │
│         ↓                                                       │
│  3. PLANIFICATION   Créer les tâches TodoWrite                  │
│         ↓                                                       │
│  4. CORRECTION      Exécuter les subagents par principe         │
│         ↓                                                       │
│  5. VALIDATION      Vérifier la non-régression                  │
└─────────────────────────────────────────────────────────────────┘
```

## Extraction du contexte (symfony-process-doc)

Quand le rapport provient de `symfony-process-doc --audit`, extraire ces sections pour enrichir les prompts des subagents :

| Section | Usage | Pour quel principe |
|---------|-------|-------------------|
| 1.1 Diagramme du flux | Comprendre les interactions | Tous |
| 4.1 Structure des fichiers | Savoir où créer les classes | DIP, SRP |
| 4.2 Interfaces (Ports) | Voir les interfaces existantes | DIP, ISP |
| 4.4 Diagramme dépendances | Relations entre classes | DIP, OCP |
| 3. Machine à états | Transitions pour Strategy | OCP |
| 7. Points d'attention | Ne pas casser l'existant | Tous |

### Étape 1 : Parser le rapport

Extraire les violations du rapport Markdown :

```
Violation {
  principe: S | O | L | I | D
  classe: string (FQCN)
  severite: HIGH | MEDIUM | LOW
  probleme: string
  recommandation: string
}
```

### Étape 2 : Prioriser

Ordre de traitement :
1. **DIP** en premier (fondation pour les autres)
2. **SRP** ensuite (simplifie les classes avant autres refactorings)
3. **ISP** (interfaces avant implémentations)
4. **OCP** (patterns après structure)
5. **LSP** (vérifications finales)

Dans chaque principe, traiter par sévérité : HIGH → MEDIUM → LOW.

### Étape 3 : Créer les tâches

Utiliser `TodoWrite` pour créer un plan structuré :

```
- [ ] [DIP-HIGH] Créer OrderRepositoryInterface dans Domain
- [ ] [SRP-HIGH] Extraire UserAuthService de UserService
- [ ] [OCP-MEDIUM] Remplacer switch par TaggedServices dans NotificationService
```

### Étape 4 : Exécuter les corrections

Lancer les subagents appropriés par type de violation.

### Étape 5 : Valider

Exécuter les tests et re-lancer un audit pour vérifier les améliorations.

## Subagents par principe

### [D] Dependency Inversion - `symfony-expert`

**Modèle recommandé** : `sonnet` (refactoring structurel)

**Tâches** :
- Créer interface dans Domain (Port)
- Déplacer l'implémentation vers Infrastructure (Adapter)
- Mettre à jour les services.yaml pour l'autowiring
- Remplacer les `new` par injection de dépendance

**Prompt type** (avec contexte architectural) :

```
Corriger la violation DIP dans `{classe}`.

## Contexte architectural

### Structure du module
{section_4_1_structure}

### Interfaces existantes (Ports)
{section_4_2_interfaces}

### Diagramme des dépendances
{section_4_4_diagram}

## Violation

**Problème** : {probleme}
**Recommandation** : {recommandation}

## Actions
1. Créer l'interface dans `src/Domain/Repository/` (voir structure ci-dessus)
2. L'implémentation existante doit implémenter cette interface
3. Mettre à jour `services.yaml` pour lier l'interface à l'implémentation
4. Remplacer tous les usages du type concret par l'interface

Respecter les conventions de `php-clean-architecture`.
```

**Prompt type** (sans contexte - fallback pour symfony-solid-audit) :

```
Corriger la violation DIP dans `{classe}`.

Problème : {probleme}
Recommandation : {recommandation}

Actions :
1. Créer l'interface `{Entity}RepositoryInterface` dans `src/Domain/Repository/`
2. L'implémentation existante doit implémenter cette interface
3. Mettre à jour `services.yaml` pour lier l'interface à l'implémentation
4. Remplacer tous les usages du type concret par l'interface

Respecter les conventions de `php-clean-architecture`.
```

### [S] Single Responsibility - `symfony-expert`

**Modèle recommandé** : `sonnet` (extraction de classes)

**Tâches** :
- Identifier les responsabilités distinctes
- Extraire en classes/services séparés
- Créer les Commands/Queries si CQRS
- Mettre à jour les dépendances

**Prompt type** (avec contexte architectural) :

```
Corriger la violation SRP dans `{classe}`.

## Contexte architectural

### Structure du module
{section_4_1_structure}

### Diagramme du flux
{section_1_1_flowchart}

### Points d'attention
{section_7_points_attention}

## Violation

**Problème** : {probleme}
**Recommandation** : {recommandation}

## Actions
1. Analyser les méthodes et identifier les groupes de responsabilités
2. Créer les nouvelles classes : {classes_a_extraire} (dans la structure existante)
3. Déplacer les méthodes correspondantes
4. Injecter les nouvelles dépendances dans la classe originale
5. Mettre à jour les usages externes

Utiliser CQRS si approprié (Command/Query handlers).
Respecter les conventions de `php-clean-architecture`.
```

**Prompt type** (sans contexte - fallback) :

```
Corriger la violation SRP dans `{classe}`.

Problème : {probleme}
Recommandation : {recommandation}

Actions :
1. Analyser les méthodes et identifier les groupes de responsabilités
2. Créer les nouvelles classes : {classes_a_extraire}
3. Déplacer les méthodes correspondantes
4. Injecter les nouvelles dépendances dans la classe originale
5. Mettre à jour les usages externes

Utiliser CQRS si approprié (Command/Query handlers).
```

### [I] Interface Segregation - `symfony-expert`

**Modèle recommandé** : `haiku` (refactoring simple)

**Tâches** :
- Découper les interfaces larges
- Créer des interfaces spécifiques par rôle
- Mettre à jour les implémentations

**Prompt type** (avec contexte architectural) :

```
Corriger la violation ISP dans `{interface}`.

## Contexte architectural

### Interfaces existantes (Ports)
{section_4_2_interfaces}

### Diagramme des dépendances
{section_4_4_diagram}

### Points d'attention
{section_7_points_attention}

## Violation

**Problème** : {probleme}
**Recommandation** : {recommandation}

## Actions
1. Identifier les groupes de méthodes par responsabilité
2. Créer les nouvelles interfaces : {interfaces_a_creer}
3. Faire hériter ou composer selon le besoin
4. Mettre à jour les implémentations
5. Mettre à jour les type hints dans les consommateurs

Respecter les conventions de `php-clean-architecture`.
```

**Prompt type** (sans contexte - fallback) :

```
Corriger la violation ISP dans `{interface}`.

Problème : {probleme}
Recommandation : {recommandation}

Actions :
1. Identifier les groupes de méthodes par responsabilité
2. Créer les nouvelles interfaces : {interfaces_a_creer}
3. Faire hériter ou composer selon le besoin
4. Mettre à jour les implémentations
5. Mettre à jour les type hints dans les consommateurs
```

### [O] Open/Closed - `symfony-expert`

**Modèle recommandé** : `sonnet` (patterns avancés)

**Tâches** :
- Remplacer switch/case par Strategy pattern
- Implémenter Tagged Services Symfony
- Créer les interfaces et implémentations

**Prompt type** (avec contexte architectural) :

```
Corriger la violation OCP dans `{classe}`.

## Contexte architectural

### Machine à états (si applicable)
{section_3_state_diagram}

### Diagramme des dépendances
{section_4_4_diagram}

### Points d'attention
{section_7_points_attention}

## Violation

**Problème** : {probleme} (switch/instanceof détecté)
**Recommandation** : {recommandation}

## Actions
1. Créer l'interface `{Strategy}Interface` avec `supports()` et méthode métier
2. Ajouter l'attribut `#[AutoconfigureTag('app.{tag}')]`
3. Créer une implémentation par cas du switch (voir états ci-dessus si applicable)
4. Injecter via `#[TaggedIterator('app.{tag}')]`
5. Supprimer le switch, itérer sur les stratégies

Respecter les conventions de `php-clean-architecture`.
Voir `references/ocp-patterns.md` pour les exemples.
```

**Prompt type** (sans contexte - fallback) :

```
Corriger la violation OCP dans `{classe}`.

Problème : {probleme} (switch/instanceof détecté)
Recommandation : {recommandation}

Actions :
1. Créer l'interface `{Strategy}Interface` avec `supports()` et méthode métier
2. Ajouter l'attribut `#[AutoconfigureTag('app.{tag}')]`
3. Créer une implémentation par cas du switch
4. Injecter via `#[TaggedIterator('app.{tag}')]`
5. Supprimer le switch, itérer sur les stratégies

Voir `references/ocp-patterns.md` pour les exemples.
```

### [L] Liskov Substitution - `symfony-expert`

**Modèle recommandé** : `haiku` (vérification/correction simple)

**Tâches** :
- Identifier les violations de contrat
- Corriger les return types
- Supprimer les exceptions inappropriées

**Prompt type** (avec contexte architectural) :

```
Corriger la violation LSP dans `{classe}`.

## Contexte architectural

### Diagramme du flux
{section_1_1_flowchart}

### Points d'attention
{section_7_points_attention}

## Violation

**Problème** : {probleme}
**Recommandation** : {recommandation}

## Actions
1. Vérifier le contrat de la classe parente/interface
2. Aligner le comportement de la méthode override
3. Ne pas renforcer les préconditions
4. Ne pas affaiblir les postconditions
5. Supprimer les `throw new \Exception` si le parent ne le fait pas

Respecter les conventions de `php-clean-architecture`.
```

**Prompt type** (sans contexte - fallback) :

```
Corriger la violation LSP dans `{classe}`.

Problème : {probleme}
Recommandation : {recommandation}

Actions :
1. Vérifier le contrat de la classe parente/interface
2. Aligner le comportement de la méthode override
3. Ne pas renforcer les préconditions
4. Ne pas affaiblir les postconditions
5. Supprimer les `throw new \Exception` si le parent ne le fait pas
```

## Orchestration multi-violations

Pour un rapport avec plusieurs violations, exécuter les subagents en parallèle quand possible :

```
┌─────────────────────────────────────────────────────────────┐
│  Parallel Execution Pool                                     │
├──────────────────┬──────────────────┬───────────────────────┤
│  DIP Fixes       │  SRP Fixes       │  ISP Fixes            │
│  (sonnet)        │  (sonnet)        │  (haiku)              │
│                  │                  │                       │
│  - Interface 1   │  - Extract Svc1  │  - Split Interface1   │
│  - Interface 2   │  - Extract Svc2  │  - Split Interface2   │
└──────────────────┴──────────────────┴───────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│  Sequential (dépend des précédents)                          │
├─────────────────────────────────────────────────────────────┤
│  OCP Fixes (sonnet) → LSP Fixes (haiku)                     │
└─────────────────────────────────────────────────────────────┘
```

## Commande d'exécution

Utiliser le Tool `Task` avec `subagent_type: symfony-expert` :

```
Task(
  description: "Fix DIP violation in OrderService",
  prompt: "<prompt structuré ci-dessus>",
  subagent_type: "symfony-expert",
  model: "sonnet"  // ou "haiku" selon complexité
)
```

## Validation post-correction

Après chaque batch de corrections :

1. **Tests unitaires** : `./vendor/bin/phpunit`
2. **Analyse statique** : `./vendor/bin/phpstan analyse`
3. **Re-audit partiel** : relancer `symfony-solid-audit` sur les fichiers modifiés

## Références

Consulter `references/correction-patterns.md` pour les patterns de correction détaillés par principe.
