---
name: launch-audit-team
description: "Orchestrate a SOLID audit and optional correction for Symfony/PHP code using coordinated agents. Scan for SRP, OCP, LSP, ISP, DIP violations, generate a scored Markdown report, and optionally fix violations with a 4-agent team (solid-scanner, dip-srp-fixer, ocp-isp-fixer, validation-agent). Use when the user wants a SOLID audit, architecture review, code quality scan, asks 'lance un audit', 'audit SOLID', 'vérifie le code', '/launch-audit-team', or wants to detect and fix SOLID violations in PHP code."
user_invocable: true
---

# Launch Audit Team

3-phase interactive flow: identify target, choose scope, launch agents.

## Phase 1 — Cible

Use `AskUserQuestion`: **"Quel code veux-tu auditer ? (répertoire, feature, ou classe spécifique)"**

Then:
1. Verify path exists with `Glob`
2. If not found → signal, ask for correction
3. If broad scope → list subdirectories, confirm perimeter
4. Reformulate target, confirm with user

## Phase 2 — Scope

Use `AskUserQuestion` with 2 options:

1. **Audit seul** — Rapport SOLID avec score et recommandations. Aucune modification.
2. **Audit + correction** — Scan puis team de correction. Confirmation avant chaque batch.

## Phase 3 — Lancement

Determine the path:

**Audit seul?** → Follow "Audit only" below
**Audit + correction?** → Follow "Full correction" below

### Audit only

Launch single subagent (no team needed):

```
Task(
  description: "SOLID audit on [cible]",
  subagent_type: "solid-scanner",
  prompt: "Audit SOLID complet sur [cible]. Rapport Markdown avec :
    - Score global /100 et par principe (S, O, L, I, D) chacun /20
    - Violations classées par sévérité (critique, haute, moyenne, basse)
    - Par violation : FQCN, principe violé, problème, recommandation
    - Rappel : entités Doctrine dans Domain ≠ violation DIP (choix projet)"
)
```

Return report to user. Done.

### Full correction

Read [references/audit-team-composition.md](references/audit-team-composition.md) for detailed agent setup, task dependencies, prompt templates, and coordination rules.

### Quick reference — Dependency chain

```
solid-scanner (first)
    └── dip-srp-fixer (after scan + user confirmation)
            └── ocp-isp-fixer (after DIP fixes + user confirmation)
validation-agent (after each fixer batch — manually coordinated by team lead)
```

### Execution Steps

1. `TeamCreate` — name: `audit-solid-{scope-name}`
2. `TaskCreate` × 4 — with `blockedBy` dependencies (see references)
3. Launch `solid-scanner` via `Task` with `team_name`, `name: "solid-scanner"`, `subagent_type: "solid-scanner"`
4. When scanner completes → extract violation counts from report
5. `AskUserQuestion`: "Le fixer va corriger [N] violations DIP+SRP. On lance ?" (Oui / Détail d'abord)
6. Launch `dip-srp-fixer` via `Task` with `team_name`, `mode: "bypassPermissions"`, `name: "dip-srp-fixer"` — pass scan report in prompt
7. After each file batch → launch `validation-agent` via `Task` with `team_name`, `name: "validation-agent"`
8. If regression → `SendMessage` to `dip-srp-fixer` with error, wait for fix
9. When DIP+SRP done → `AskUserQuestion`: "Le fixer va corriger [N] violations OCP+ISP+LSP. On lance ?"
10. Launch `ocp-isp-fixer` via `Task` with `team_name`, `mode: "bypassPermissions"`, `name: "ocp-isp-fixer"` — pass relevant violations
11. Same validation loop as step 7-8
12. Final: launch `validation-agent` for re-audit of modified files → score comparison

### User Checkpoints

Ask confirmation at key milestones:

- **After scan** — `AskUserQuestion`: "[N] violations DIP+SRP détectées. On lance les corrections ?" (Oui / Détail d'abord)
- **After DIP+SRP** — `AskUserQuestion`: "[N] violations OCP+ISP+LSP restantes. On continue ?" (Oui / Je review d'abord)

### Completion and Shutdown

1. Verify ALL tasks are marked `completed` via `TaskList`
2. Verify `validation-agent` reports all tests passing (PHPUnit + PHPStan green)
3. Present final report to user: score comparison (before → after), corrections applied, regressions
4. Shut down ALL agents: `SendMessage` with `type: "shutdown_request"` to each agent
5. After all agents have shut down → `TeamDelete` to clean up

## Critical Invariants

- **Scanner first**: NO fixer starts before `solid-scanner` completes
- **DIP before OCP**: `ocp-isp-fixer` MUST NOT start before `dip-srp-fixer` completes
- **Regression = STOP**: If `validation-agent` reports failure → `SendMessage` to responsible fixer with file, line, error. Fixer MUST fix before continuing.
- **Iso-functional**: All corrections preserve existing behavior — zero feature changes
- **services.yaml**: Modified ONLY by `dip-srp-fixer`
- **Confirmation gates**: Always ask user before each correction phase
- **Shutdown**: After all tasks completed, send `shutdown_request` to each agent, then `TeamDelete`
