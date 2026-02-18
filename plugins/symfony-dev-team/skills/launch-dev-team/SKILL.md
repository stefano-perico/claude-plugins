---
name: launch-dev-team
description: "Orchestrate the launch of a Symfony feature development Agent Team with 5 specialized agents (domain-architect, app-orchestrator, infra-implementor, test-guardian, api-exposer) following hexagonal architecture and CQRS patterns. Use when developing a new Symfony feature as a team, when the user says 'lance une team dev', 'développe cette feature en team', '/launch-dev-team', or wants to build a feature with multiple coordinated agents."
user_invocable: true
---

# Launch Dev Team

Orchestrate a 3-phase interactive flow to launch a Symfony feature development team.

## Process Overview

1. **Cadrage** — Understand the feature
2. **Approche** — Choose spec / plan / direct
3. **Lancement** — Create the Agent Team with proper dependency chain

## Phase 1 — Cadrage

Use `AskUserQuestion` to ask: **"Quelle feature veux-tu développer ?"**

After the response, reformulate in 3-5 lines covering:
- Business objective
- Key entities/concepts
- Main interactions (API, events, external services)

Confirm understanding with `AskUserQuestion`: "Ma compréhension est-elle correcte ?" (Oui / Non, je précise)

## Phase 2 — Approche

Use `AskUserQuestion` with 3 options:

1. **Spec d'abord** — Rédiger une spec structurée (entités, flux, events, contrats d'interface) puis lancer la team. Plus fiable pour features complexes.
2. **Plan rapide** — Plan mode rapide (structure de fichiers + répartition) puis lancer la team. Bon compromis.
3. **Direct** — Lancer la team immédiatement. Pour features simples et bien comprises uniquement.

## Phase 3 — Lancement

Determine the path:

**Spec d'abord?** → Follow "Spec d'abord" below
**Plan rapide?** → Follow "Plan rapide" below
**Direct?** → Follow "Direct" below

### Option "Spec d'abord"

1. Enter plan mode via `EnterPlanMode`
2. Draft structured spec: aggregates, entities, VOs, domain events, repository interfaces, Commands/Queries, API endpoints, business rules
3. Validate with user via `ExitPlanMode`
4. Create Agent Team, pass spec as context to all agents

### Option "Plan rapide"

1. Enter plan mode via `EnterPlanMode`
2. Produce file list per layer (Domain, Application, Infrastructure, API) + agent assignments
3. Validate with user via `ExitPlanMode`
4. Create Agent Team

### Option "Direct"

Create Agent Team immediately with user's feature description.

## Team Creation

Read [references/team-composition.md](references/team-composition.md) for detailed agent composition, task dependencies, prompt templates, and coordination rules.

### Quick reference — Dependency chain

```
domain-architect (first)
    ├── app-orchestrator (after Domain)
    ├── infra-implementor (after Domain, parallel with app-orchestrator)
    └── test-guardian (after Domain, continuous validation)
            └── api-exposer (after Application)
```

### Execution Steps

1. `TeamCreate` with descriptive name (e.g., `dev-feature-{feature-name}`)
2. `TaskCreate` x 5 — with `blockedBy` dependencies (see references)
3. Launch `domain-architect` via `Task` with `team_name`, `mode: "bypassPermissions"`, `subagent_type: "domain-architect"`
4. When domain-architect completes → launch `app-orchestrator`, `infra-implementor`, `test-guardian` in parallel
5. Pass domain interfaces/contracts to downstream agents via their prompts
6. When app-orchestrator completes → launch `api-exposer`
7. Use `SendMessage` to relay interface changes or regressions between agents
8. Monitor via `TaskList` and incoming agent messages

### User Checkpoints

Ask confirmation at key milestones:

- **After Domain** — `AskUserQuestion`: "Le Domain est posé. On lance Application + Infrastructure + Tests ?" (Oui / Je review d'abord)
- **After Application** — `AskUserQuestion`: "La couche Application est prête. On lance l'exposition API ?" (Oui / Je review d'abord)

### Completion and Shutdown

1. Verify ALL tasks are marked `completed` via `TaskList`
2. Verify `test-guardian` reports all tests passing (PHPUnit + PHPStan green)
3. Present a summary to the user: files created per layer, test results
4. Shut down ALL agents: `SendMessage` with `type: "shutdown_request"` to each agent
5. After all agents have shut down → `TeamDelete` to clean up

## Critical Invariants

- **Domain first**: NO agent starts before `domain-architect` completes
- **Application before API**: `api-exposer` MUST NOT start before `app-orchestrator` completes
- **Regression = STOP**: If `test-guardian` reports failure → `SendMessage` to responsible agent with file, line, error. Agent MUST fix before continuing.
- **services.yaml**: Modified ONLY by `infra-implementor`
- **Interface changes**: If `domain-architect` modifies an interface after others started, immediately notify ALL dependent agents via `SendMessage`
- **Conflicts**: If ambiguity arises, ask the user via `AskUserQuestion`
