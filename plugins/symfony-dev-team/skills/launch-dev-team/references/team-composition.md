# Dev Team Composition

## Table of Contents

1. [Agents](#agents)
2. [Task Setup](#task-setup)
3. [Agent Launch Configuration](#agent-launch-configuration)
4. [Prompt Templates](#prompt-templates)
5. [Validation Loop](#validation-loop)
6. [Coordination Rules](#coordination-rules)

## Agents

| Agent | subagent_type | Role | Starts when |
|-------|--------------|------|-------------|
| domain-architect | `domain-architect` | Entities, VOs, events, ports, exceptions in `src/Domain/` | Immediately |
| app-orchestrator | `app-orchestrator` | Commands, Queries, Handlers, DTOs in `src/Application/` | After Domain done |
| infra-implementor | `infra-implementor` | Doctrine repos, adapters, services.yaml in `src/Infrastructure/` | After Domain done (parallel with app-orchestrator) |
| test-guardian | `test-guardian` | Unit/integration/functional tests + PHPUnit/PHPStan validation | After Domain done (continuous) |
| api-exposer | `api-exposer` | ApiPlatform Resources, Providers, Processors | After Application done |

## Task Setup

Create 5 tasks with these exact dependencies:

```
Task 1: "Modéliser le Domain de [feature]"
  → blockedBy: none
  → owner: domain-architect

Task 2: "Câbler la couche Application CQRS de [feature]"
  → blockedBy: [Task 1]
  → owner: app-orchestrator

Task 3: "Implémenter l'Infrastructure de [feature]"
  → blockedBy: [Task 1]
  → owner: infra-implementor

Task 4: "Écrire les tests et valider en continu [feature]"
  → blockedBy: [Task 1]
  → owner: test-guardian

Task 5: "Exposer l'API de [feature]"
  → blockedBy: [Task 2]
  → owner: api-exposer
```

## Agent Launch Configuration

Launch ALL agents via `Task` with these parameters:

```
Task(
  subagent_type: "<agent-type>",
  team_name: "dev-feature-{name}",
  name: "<agent-name>",
  mode: "bypassPermissions",
  prompt: "<see prompt templates below>"
)
```

**IMPORTANT**: Always set `mode: "bypassPermissions"` — agents write code and must not be blocked by permission prompts.

## Prompt Templates

### domain-architect

```
Feature: [description]
[Spec or plan if produced in Phase 2]

Modélise le Domain pour cette feature :
- Entités et Value Objects dans src/Domain/[Feature]/
- Domain Events
- Repository interfaces (ports)
- Exceptions métier
- Business rules et invariants

Conventions : entités Doctrine dans le Domain (annotations ORM acceptées — choix projet).
Quand tu as terminé, marque ta task comme completed.
```

### app-orchestrator

```
Feature: [description]
[Spec or plan if produced in Phase 2]

Interfaces et contrats du Domain :
[paste output from domain-architect — list of interfaces, entities, events]

Câble la couche Application CQRS :
- Commands + CommandHandlers dans src/Application/[Feature]/Command/
- Queries + QueryHandlers dans src/Application/[Feature]/Query/
- DTOs si nécessaire
- Wiring Symfony Messenger (sync par défaut)

Quand tu as terminé, marque ta task comme completed.
```

### infra-implementor

```
Feature: [description]
[Spec or plan if produced in Phase 2]

Interfaces à implémenter :
[paste repository interfaces and ports from domain-architect]

Implémente la couche Infrastructure :
- Repositories Doctrine dans src/Infrastructure/[Feature]/Repository/
- Adapters pour services externes dans src/Infrastructure/[Feature]/Adapter/
- Event subscribers si nécessaire
- Configuration services.yaml (tu es le SEUL à modifier ce fichier)

Quand tu as terminé, marque ta task comme completed.
```

### test-guardian

```
Feature: [description]
[Spec or plan if produced in Phase 2]

Classes Domain créées :
[paste list of domain classes from domain-architect]

Écris les tests et valide en continu :
- Tests unitaires Domain (VOs, Entities, business rules)
- Tests intégration (Handlers, Repositories)
- Tests fonctionnels API (quand api-exposer aura fini)
- Lance PHPUnit et PHPStan après chaque batch de tests

Si une régression est détectée, envoie immédiatement un message au team lead
avec : fichier, ligne, message d'erreur, agent responsable probable.

Quand TOUS les tests passent, marque ta task comme completed.
```

### api-exposer

```
Feature: [description]
[Spec or plan if produced in Phase 2]

Commands et Queries disponibles :
[paste list of commands/queries from app-orchestrator]

Expose l'API via ApiPlatform :
- Resources dans src/Infrastructure/[Feature]/ApiPlatform/Resource/
- State Providers
- State Processors (thin — max 30 lignes, routent vers Commands/Queries)

Quand tu as terminé, marque ta task comme completed.
```

## Validation Loop

`test-guardian` runs continuously after Domain is ready.

### After each agent batch

1. `test-guardian` runs `PHPUnit` + `PHPStan`
2. **Pass** → confirm to team lead: "Batch validé, aucune régression"
3. **Fail** → message to team lead with:
   - Failing test / PHPStan error
   - File and line number
   - Probable responsible agent

### Team lead on regression

1. Receive failure message from `test-guardian`
2. Forward to responsible agent via `SendMessage`:
   `"Régression détectée : {file}:{line} — {error}. Corrige avant de continuer."`
3. Wait for fix confirmation before unblocking dependent work

### Final validation

After ALL agents complete:
1. `test-guardian` runs full test suite + PHPStan
2. Reports final status to team lead
3. Team lead presents results to user

## Coordination Rules

- **services.yaml**: Modified ONLY by `infra-implementor`
- **Interface changes**: If `domain-architect` modifies an interface after downstream agents started, team lead MUST immediately notify ALL dependent agents via `SendMessage`
- **Regressions**: When `test-guardian` reports a failure, team lead forwards to the responsible agent with file, line, error detail. Agent MUST fix before continuing.
- **Completion**: ALL agents must finish AND all tests must pass before returning control to user
- **Conflicts**: If ambiguity arises, team lead asks the user via `AskUserQuestion`
- **Shutdown**: After all tasks completed, send `shutdown_request` to each agent, then `TeamDelete`
