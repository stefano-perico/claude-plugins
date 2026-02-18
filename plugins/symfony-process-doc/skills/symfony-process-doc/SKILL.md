---
name: symfony-process-doc
description: Génération de documentation technique de processus Symfony avec diagrammes Mermaid (flowchart, sequence, class, state). Analyser un workflow, une saga, un service ou un handler pour produire un document .md structuré documentant l'architecture, les interactions entre classes, et les flux de données. Option d'audit SOLID intégré. Déclencher quand l'utilisateur demande de documenter un process, générer une doc technique, créer un flowchart d'architecture, ou documenter un workflow Symfony.
---

# Documentation de Processus Symfony

Générer une documentation technique complète au format Markdown avec diagrammes Mermaid pour documenter un processus, workflow, saga ou feature Symfony.

## Paramètres

| Paramètre | Description | Exemple |
|-----------|-------------|---------|
| `cible` | Chemin du répertoire ou fichier à documenter | `src/SendDeclaration/` |
| `output` | Chemin du fichier .md de sortie | `docs/saga-envoi.md` |
| `--audit` | Active l'audit SOLID (optionnel) | Flag sans valeur |

## Architecture d'exécution

Ce skill utilise une architecture de **subagents spécialisés** pour optimiser les coûts et la qualité.

```
┌─────────────────────────────────────────────────────────────────┐
│ Phase 1 (séquentielle)                                          │
│   Scanner (haiku) ──→ Analyzer (sonnet)                         │
├─────────────────────────────────────────────────────────────────┤
│ Phase 2 (parallèle)                                             │
│   ├── Mermaid (haiku)                                           │
│   └── SOLID (sonnet) ← si --audit                               │
├─────────────────────────────────────────────────────────────────┤
│ Phase 3 (séquentielle)                                          │
│   Assembler (haiku) ──→ Write .md                               │
└─────────────────────────────────────────────────────────────────┘
```

### Subagents

| Agent | Modèle | Fichier | Rôle |
|-------|--------|---------|------|
| Scanner | **haiku** | `~/.claude/agents/process-doc-scanner.md` | Glob/Grep fichiers, détecte patterns |
| Analyzer | **sonnet** | `~/.claude/agents/process-doc-analyzer.md` | Analyse relations, flux, dépendances |
| Mermaid | **haiku** | `~/.claude/agents/process-doc-mermaid.md` | Génère diagrammes Mermaid |
| SOLID | **sonnet** | `~/.claude/agents/process-doc-solid.md` | Audit SOLID (optionnel) |
| Assembler | **haiku** | `~/.claude/agents/process-doc-assembler.md` | Assemble le document final |

**Orchestrateur** : `~/.claude/agents/process-doc-orchestrator.md`

### Coût optimisé

```
Sans --audit : 3× haiku + 1× sonnet (~60% économie vs tout sonnet)
Avec --audit : 3× haiku + 2× sonnet (~50% économie vs tout sonnet)
```

## Workflow détaillé

### 1. Scanner (haiku)

Lister fichiers PHP et identifier :
- Type : Saga, Workflow, Handler, Service, Entity
- Structure : Domain/, Application/, Infrastructure/
- Patterns : CQRS, Hexagonal, Event-driven

### 2. Analyzer (sonnet)

Analyser en profondeur :
- Relations entre classes (dépendances, héritages)
- Flux d'exécution (Command → Handler → Service)
- Machine à états (enums, transitions)
- Interfaces et implémentations

### 3. Mermaid (haiku)

Générer les diagrammes :
- Flowchart du flux principal
- StateDiagram de la machine à états
- SequenceDiagram des interactions
- ClassDiagram des dépendances

### 4. SOLID (sonnet, optionnel)

Si `--audit` : analyser les 5 principes SOLID et calculer un score.

### 5. Assembler (haiku)

Assembler le document final selon le template.

## Structure du document généré

```markdown
# [Nom du Process] - Documentation

> **Date** : YYYY-MM-DD
> **Stack** : PHP X.X, Symfony X.X, [technologies]
> **Architecture** : [patterns utilisés]

## 1. Vue d'ensemble
## 2. Composants
## 3. Machine à états (si applicable)
## 4. Architecture (Ports & Adapters)
## 5. Entités et modèles
## 6. Repository et persistance
## 7. Points d'attention
## 8. Audit SOLID (si --audit)
## 9. Tests
## 10. Annexes
```

## Patterns à détecter

| Pattern | Indicateurs |
|---------|-------------|
| **CQRS** | `Command`, `Query`, `Handler` suffixes |
| **Saga** | `AbstractSaga`, `SagaStatus`, steps |
| **Event-driven** | `Event` suffix, `EventSubscriber`, Messenger |
| **Hexagonal** | `Domain/Port`, `Infrastructure/Adapter` |
| **Repository** | `RepositoryInterface`, Doctrine entities |

## Références

- `references/document-template.md` : template complet du document généré
- `references/mermaid-patterns.md` : exemples de diagrammes par type de processus
- `~/.claude/agents/process-doc-*.md` : définitions des subagents
