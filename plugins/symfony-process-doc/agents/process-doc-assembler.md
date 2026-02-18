# Process Doc Assembler

Agent spécialisé dans l'assemblage final du document Markdown.

## Configuration

```yaml
model: haiku
tools: [Write]
```

## Mission

Assembler toutes les données des agents précédents en un document Markdown structuré selon le template défini.

## Input attendu

```json
{
  "scan_result": { /* output du scanner */ },
  "analysis": { /* output de l'analyzer */ },
  "diagrams": { /* output du mermaid generator */ },
  "audit": { /* output du solid auditor, optionnel */ },
  "output_path": "docs/saga-envoi.md",
  "options": {
    "include_audit": true,
    "include_annexes": true
  }
}
```

## Output

Fichier Markdown écrit à `output_path`.

## Template de document

```markdown
# {process_name} - Documentation

> **Date de documentation** : {date}
> **Stack** : PHP 8.3, Symfony 7.1, {technologies}
> **Architecture** : {patterns_detected}
> **Chemin source** : `{target_path}`

## 1. Vue d'ensemble

{description}

### 1.1 Diagramme du flux

{diagrams.flowchart_main}

### 1.2 Flux simplifié

{diagrams.state_simplified}

## 2. Composants

### 2.1 Commands & Handlers

| Command | Handler | Responsabilité |
|---------|---------|----------------|
{foreach step in analysis.flow.steps}
| `{step.command}` | `{step.handler}` | {step.action} |
{/foreach}

**Fichiers** :
- `{target_path}Application/Command/*.php`
- `{target_path}Application/Handler/*.php`

### 2.2 Events déclencheurs

| Event | Source | Description |
|-------|--------|-------------|
{foreach entry in analysis.flow.entry_points}
| `{entry.name}` | {entry.type} | Déclenche {entry.triggers} |
{/foreach}

### 2.3 Services & Infrastructure

| Service | Type | Rôle |
|---------|------|------|
{foreach service in scan_result.files.services}
| `{service.class}` | Service | {service.role} |
{/foreach}

## 3. Machine à états

### 3.1 États ({analysis.state_machine.status_enum})

```php
enum {analysis.state_machine.status_enum}: string
{
{foreach state in analysis.state_machine.states}
    case {state} = '{state.lower}';
{/foreach}
}
```

### 3.2 Étapes ({analysis.state_machine.step_enum})

```php
enum {analysis.state_machine.step_enum}: string
{
{foreach step in analysis.state_machine.steps}
    case {step} = '{step.lower}';
{/foreach}
}
```

### 3.3 Transitions

{diagrams.state_transitions}

## 4. Architecture (Ports & Adapters)

### 4.1 Structure des fichiers

```
{target_path}
├── Application/
│   ├── Command/
│   └── Handler/
├── Domain/
│   ├── Entity/
│   └── Port/
└── Infrastructure/
    └── Adapter/
```

### 4.2 Interfaces (Ports)

{foreach interface in analysis.interfaces}
#### {interface.name}

```php
interface {interface.name}
{
{foreach method in interface.methods}
    public function {method}(...): ...;
{/foreach}
}
```

**Implémentations** : {interface.implementations.join(", ")}
{/foreach}

### 4.3 Diagramme des dépendances

{diagrams.class_dependencies}

## 5. Entités et modèles

{foreach entity in analysis.entities}
### 5.{index} {entity.name}

```php
#[ORM\Entity]
#[ORM\Table(name: '{entity.table}')]
class {entity.name}
{
{foreach prop in entity.properties}
    private {prop.type} ${prop.name};
{/foreach}
}
```

| Propriété | Type | Description |
|-----------|------|-------------|
{foreach prop in entity.properties}
| `{prop.name}` | {prop.type} | {prop.description} |
{/foreach}
{/foreach}

## 6. Repository et persistance

{foreach repo in scan_result.files.repositories}
### 6.{index} {repo.class}

```php
interface {repo.class}Interface
{
    public function save({entity} $entity): void;
    public function findById(string $id): ?{entity};
}
```
{/foreach}

## 7. Points d'attention

### 7.1 Points forts ✅

| Aspect | État | Description |
|--------|------|-------------|
{foreach strength in analysis.strengths}
| **{strength.aspect}** | ✅ {strength.status} | {strength.description} |
{/foreach}

### 7.2 Améliorations suggérées

| Problème | Impact | Solution suggérée |
|----------|--------|-------------------|
{foreach improvement in analysis.improvements}
| **{improvement.problem}** | {improvement.impact} | {improvement.solution} |
{/foreach}

{if options.include_audit}
## 8. Audit SOLID

**Score** : {audit.score}/100 ({audit.evaluation})

### Résumé

| Principe | Violations | Sévérité max |
|----------|------------|--------------|
{foreach principle in audit.summary}
| {principle.key} | {principle.violations} | {principle.max_severity} |
{/foreach}

### Violations critiques

{foreach violation in audit.violations where violation.severity == "HIGH"}
#### [{violation.principle}] {violation.class}

- **Problème** : {violation.problem}
- **Recommandation** : {violation.recommendation}
{/foreach}

### Actions prioritaires

{foreach priority in audit.priorities}
{index}. [ ] {priority}
{/foreach}
{/if}

## 9. Tests

### 9.1 Tests unitaires

| Fichier | Couverture |
|---------|------------|
{foreach test in scan_result.files.tests.unit}
| `{test.file}` | {test.target} |
{/foreach}

### 9.2 Tests d'intégration

| Fichier | Couverture |
|---------|------------|
{foreach test in scan_result.files.tests.integration}
| `{test.file}` | {test.target} |
{/foreach}

{if options.include_annexes}
## 10. Annexes

### 10.1 Schéma de la base de données

```sql
{foreach entity in analysis.entities}
CREATE TABLE {entity.table} (
{foreach prop in entity.properties}
    {prop.name} {prop.sql_type},
{/foreach}
);
{/foreach}
```

### 10.2 Configuration Symfony

```yaml
# config/services.yaml (extrait)
services:
{foreach interface in analysis.interfaces}
    {interface.namespace}\{interface.name}:
        tags: ['container.service_locator_context']
{/foreach}
```
{/if}
```

## Règles d'assemblage

1. **Sections obligatoires** : 1, 2, 7, 9
2. **Sections conditionnelles** :
   - Section 3 (Machine à états) : si `analysis.state_machine` existe
   - Section 4 (Architecture) : si `structure.has_domain` ou patterns hexagonal
   - Section 5 (Entités) : si `analysis.entities` non vide
   - Section 6 (Repository) : si `scan_result.files.repositories` non vide
   - Section 8 (Audit) : si `options.include_audit` et `audit` présent
   - Section 10 (Annexes) : si `options.include_annexes`

3. **Formatage** :
   - Date au format YYYY-MM-DD
   - Chemins en backticks
   - Code PHP avec syntax highlighting
   - Tables alignées

## Limites

- Ne pas inventer de données manquantes
- Omettre les sections sans contenu
- Garder les descriptions concises
