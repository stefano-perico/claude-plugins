# Audit Team Composition

## Table of Contents

1. [Agents](#agents)
2. [Task Setup](#task-setup)
3. [Agent Launch Configuration](#agent-launch-configuration)
4. [Prompt Templates](#prompt-templates)
5. [Correction Phases](#correction-phases)
6. [Coordination Rules](#coordination-rules)
7. [Validation Workflow](#validation-workflow)
8. [Report Format](#report-format)
9. [Shutdown and Cleanup](#shutdown-and-cleanup)

## Agents

| Agent | subagent_type | Role | Starts when |
|-------|--------------|------|-------------|
| solid-scanner | `solid-scanner` | Full SOLID scan, scored report | Immediately |
| dip-srp-fixer | `dip-srp-fixer` | Fix DIP (interfaces/ports), then SRP (extract services) | After scan report + user confirmation |
| ocp-isp-fixer | `ocp-isp-fixer` | Fix OCP (Strategy + tagged services), ISP (split interfaces), LSP | After DIP fixes + user confirmation |
| validation-agent | `validation-agent` | PHPUnit + PHPStan after each batch, final re-audit | After each fixer batch (launched by team lead) |

## Task Setup

Create 4 tasks with these exact dependencies:

```
Task 1: "Scanner [cible] pour violations SOLID"
  → blockedBy: none
  → owner: solid-scanner

Task 2: "Corriger les violations DIP et SRP"
  → blockedBy: [Task 1]
  → owner: dip-srp-fixer

Task 3: "Corriger les violations OCP, ISP et LSP"
  → blockedBy: [Task 2]
  → owner: ocp-isp-fixer

Task 4: "Valider les corrections et produire le rapport final"
  → blockedBy: [Task 2, Task 3]
  → owner: validation-agent
```

## Agent Launch Configuration

Launch ALL agents via `Task` with these parameters:

```
Task(
  description: "<short description>",
  subagent_type: "<agent-type>",
  team_name: "audit-solid-{scope-name}",
  name: "<agent-name>",
  mode: "bypassPermissions",
  prompt: "<see prompt templates below>"
)
```

**IMPORTANT**: Always set `mode: "bypassPermissions"` for fixer agents — they write code and must not be blocked by permission prompts. Scanner and validation-agent are read-only, `mode` is optional for them.

**IMPORTANT**: Always set `name` to the agent name from the table above — this is required for `SendMessage` coordination.

## Prompt Templates

### solid-scanner

```
Audit SOLID complet sur [cible].

Rapport Markdown avec :
- Score global /100 et par principe (S, O, L, I, D) chacun /20
- Violations classées par sévérité (critique, haute, moyenne, basse)
- Par violation : FQCN, principe violé, problème, recommandation
- Violations ordonnées par priorité de correction : DIP → SRP → OCP → ISP → LSP

Rappel : entités Doctrine dans Domain ≠ violation DIP (choix projet).

Quand tu as terminé, marque ta task comme completed.
```

### dip-srp-fixer

```
Rapport d'audit SOLID :
[coller le rapport complet du scanner]

Corrige les violations DIP et SRP identifiées dans ce rapport.

Ordre impératif : DIP d'abord (les interfaces doivent exister), puis SRP.

Pour chaque violation :
1. Lire le fichier source concerné
2. Appliquer la correction
3. Vérifier la cohérence avec les fichiers liés

Le services.yaml est ta responsabilité exclusive.
Les corrections doivent être iso-fonctionnelles (zéro changement de comportement).

Quand tu as terminé, marque ta task comme completed.
```

### ocp-isp-fixer

```
Rapport d'audit SOLID (violations OCP, ISP, LSP uniquement) :
[coller les sections OCP, ISP, LSP du rapport]

Interfaces DIP créées par le précédent fixer :
[lister les nouvelles interfaces créées]

Corrige les violations OCP, ISP et LSP identifiées.

Les corrections doivent être iso-fonctionnelles (zéro changement de comportement).
Ne modifie PAS services.yaml (responsabilité du dip-srp-fixer).

Quand tu as terminé, marque ta task comme completed.
```

### validation-agent

**Après chaque batch de corrections :**

```
Un batch de corrections [DIP+SRP / OCP+ISP+LSP] vient d'être appliqué sur [cible].

Lance PHPUnit et PHPStan.
Si régression → signale immédiatement avec fichier, ligne, erreur.
Si tout passe → confirme "Batch validé, aucune régression".

Marque ta task comme completed quand tu as terminé.
```

**Pour la validation finale :**

```
Toutes les corrections SOLID ont été appliquées sur [cible].

1. Lance PHPUnit + PHPStan (suite complète)
2. Re-lance un audit SOLID partiel sur les fichiers modifiés
3. Produis le rapport de synthèse avec score comparison (avant → après)

Format du rapport : voir section "Final validation output" ci-dessous.

Marque ta task comme completed quand tu as terminé.
```

## Correction Phases

### Phase 1: Scan

1. Launch `solid-scanner` on target
2. Wait for complete report with scores
3. Extract violation counts per principle from report
4. Mark Task 1 as completed

**Passing the report to fixers**: Copy the full scanner report text into the fixer's `prompt` parameter when launching them.

### Phase 2a: DIP + SRP corrections

1. Ask user confirmation: "Le fixer va corriger [N] violations DIP+SRP. On lance ?"
2. If user wants detail → show violations from report, ask again
3. Launch `dip-srp-fixer` with scan report in prompt context
4. After each file batch → launch `validation-agent`:
   - If PHPUnit/PHPStan fails → `SendMessage` to `dip-srp-fixer` with error details, STOP
   - If passes → continue
5. When all DIP+SRP done → mark Task 2 as completed

### Phase 2b: OCP + ISP + LSP corrections

1. Ask user confirmation: "Le fixer va corriger [N] violations OCP+ISP+LSP. On lance ?"
2. Launch `ocp-isp-fixer` with relevant violations from report + list of new DIP interfaces
3. Same validation loop as Phase 2a
4. When done → mark Task 3 as completed

### Phase 3: Final validation

1. Launch `validation-agent` for final pass:
   - Re-run SOLID audit on modified files only
   - Compute score comparison: initial → final
2. Mark Task 4 as completed
3. Present synthesis to user

## Coordination Rules

- **services.yaml**: Modified ONLY by `dip-srp-fixer`
- **Iso-functional**: All corrections preserve existing behavior — zero feature changes
- **Regression = STOP**: If `validation-agent` detects test failure, immediately `SendMessage` to active fixer: `"Régression détectée : {file}:{line} — {error}. Corrige avant de continuer."` Wait for fix before resuming.
- **Confirmation gates**: Always ask user before each correction phase
- **Report sharing**: Copy scanner report into fixer prompts. Do NOT rely on task descriptions alone — the full report must be in the prompt.
- **Completion**: Return control only after final report with score comparison

## Validation Workflow

### After each fixer batch

```bash
./vendor/bin/phpunit
./vendor/bin/phpstan analyse
```

- Failure → `SendMessage` to responsible fixer: `"Régression détectée : {file}:{line} — {error}"`
- Pass → `SendMessage` to team lead: `"Batch validé, aucune régression"`

### Final validation

1. Re-run SOLID audit on modified files
2. Produce synthesis report (see format below)

## Report Format

### Scanner output (consumed by fixers)

```markdown
# SOLID Audit Report — [cible]

## Score: [X]/100

| Principle | Score | Violations |
|-----------|-------|------------|
| SRP       | X/20  | N          |
| OCP       | X/20  | N          |
| LSP       | X/20  | N          |
| ISP       | X/20  | N          |
| DIP       | X/20  | N          |

## Violations

### DIP (fix first)
- **[FQCN]** — Severity: [critical/high/medium/low]
  Problem: [description]
  Recommendation: [action]

### SRP (fix second)
[same format]

### OCP (fix third)
[same format]

### ISP (fix fourth)
[same format]

### LSP (fix fifth)
[same format]
```

Violations ordered by fix priority: DIP → SRP → OCP → ISP → LSP.

### Final validation output

```markdown
# Correction Report — [cible]

## Score Comparison
| Principle | Before | After | Delta |
|-----------|--------|-------|-------|
| SRP       | X/20   | Y/20  | +Z    |
| OCP       | X/20   | Y/20  | +Z    |
| LSP       | X/20   | Y/20  | +Z    |
| ISP       | X/20   | Y/20  | +Z    |
| DIP       | X/20   | Y/20  | +Z    |
| **Total** | X/100  | Y/100 | +Z    |

## Corrections Applied
- [N] DIP violations fixed
- [N] SRP violations fixed
- [N] OCP violations fixed
- [N] ISP violations fixed
- [N] LSP violations fixed

## Remaining Violations
[list if any]

## Regressions
[none / list if any]

## Test Results
- PHPUnit: [pass/fail]
- PHPStan: [pass/fail]
```

## Shutdown and Cleanup

After all tasks completed and final report presented:

1. Send `shutdown_request` to each agent via `SendMessage`:
   - `SendMessage(type: "shutdown_request", recipient: "solid-scanner", content: "Audit terminé")`
   - `SendMessage(type: "shutdown_request", recipient: "dip-srp-fixer", content: "Corrections terminées")`
   - `SendMessage(type: "shutdown_request", recipient: "ocp-isp-fixer", content: "Corrections terminées")`
   - `SendMessage(type: "shutdown_request", recipient: "validation-agent", content: "Validation terminée")`
2. Wait for all shutdown confirmations
3. `TeamDelete` to clean up team resources
