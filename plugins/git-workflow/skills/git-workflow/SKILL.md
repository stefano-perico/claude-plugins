---
name: git-workflow
description: |
  Gestion complète du workflow Git avec commits conventionnels, branches et Pull Requests.
  Utiliser ce skill pour : (1) Créer des commits avec format Conventional Commits et scope obligatoire,
  (2) Créer et nommer des branches selon la convention feature/*/fix/*/hotfix/*,
  (3) Générer des Pull Requests avec Summary + Test Plan,
  (4) Valider le format des commits avant soumission,
  (5) Générer un changelog à partir des commits.
  Déclencher quand l'utilisateur demande : commit, committer, git commit, PR, pull request, créer une PR,
  ouvrir une PR, branche, créer une branche, nouvelle branche, changelog, générer un changelog,
  review, code review, pousser, push, git push, merger, merge, rebase, git log, historique git,
  conventional commits, nommage de branche, message de commit, "committe", "fais un commit",
  "pousse le code", "crée la PR", "prépare la PR", "fais une branche", ou toute opération Git.
---

# Git Workflow

Skill pour standardiser les interactions Git avec Conventional Commits en français.

## Commits Conventionnels

### Format obligatoire

```
<type>(<scope>): <description>

[corps optionnel]
```

### Types autorisés

| Type | Description | Exemple |
|------|-------------|---------|
| `feat` | Nouvelle fonctionnalité | `feat(api): ajouter endpoint utilisateurs` |
| `fix` | Correction de bug | `fix(auth): corriger expiration token` |
| `docs` | Documentation uniquement | `docs(readme): mettre à jour installation` |
| `style` | Formatage, pas de changement de code | `style(lint): appliquer prettier` |
| `refactor` | Refactoring sans changement fonctionnel | `refactor(user): extraire validation` |
| `test` | Ajout ou modification de tests | `test(api): ajouter tests unitaires` |
| `chore` | Maintenance, dépendances | `chore(deps): mettre à jour symfony` |

### Règles de validation

1. **Scope obligatoire** - Toujours spécifier le scope entre parenthèses (voir [scopes.md](references/scopes.md) pour la liste recommandée)
2. **Description en français** - Commencer par un verbe à l'infinitif
3. **Pas de point final** - La description ne termine pas par un point
4. **72 caractères max** - Pour la première ligne

### Workflow commit (scope par scope)

Par défaut, **créer un commit séparé pour chaque scope** afin d'obtenir un historique atomique et faciliter les reverts.

```bash
# 1. Lister les fichiers modifiés
git status

# 2. Identifier les scopes distincts parmi les fichiers modifiés
# Exemple: src/User/ → scope "user", src/Auth/ → scope "auth"

# 3. Pour CHAQUE scope, créer un commit dédié :
git add <fichiers-du-scope>
git commit -m "$(cat <<'EOF'
<type>(<scope>): <description>
EOF
)"

# 4. Répéter pour chaque scope restant
```

### Fichiers à exclure des commits

**Ne jamais committer la documentation IA** :
- `CLAUDE.md`, `AGENTS.md`
- `openspec/`, `docs/` (sauf demande explicite)
- `.claude/`

```bash
# Vérifier qu'aucun fichier IA n'est stagé
git status | grep -E "(CLAUDE|AGENTS|openspec|\.claude)"

# Si présents, les unstage
git restore --staged CLAUDE.md AGENTS.md openspec/ .claude/ docs/
```

### Format du message de commit

Le message de commit contient **uniquement** le type, scope et description. Pas de co-auteur, pas de signature IA.

```bash
# Correct
git commit -m "feat(user): ajouter validation email"

# Incorrect - ne pas ajouter de mention IA
git commit -m "feat(user): ajouter validation email

Co-Authored-By: Claude <...>"  # NON
```

## Branches

### Convention de nommage

```
<type>/<description-kebab-case>
```

### Types de branches

| Préfixe | Usage | Exemple |
|---------|-------|---------|
| `feature/` | Nouvelle fonctionnalité | `feature/ajout-export-pdf` |
| `fix/` | Correction de bug | `fix/validation-email` |
| `hotfix/` | Correction urgente prod | `hotfix/security-patch` |
| `refactor/` | Refactoring | `refactor/service-user` |
| `docs/` | Documentation | `docs/api-reference` |

### Workflow création de branche

```bash
# 1. Partir de la branche principale
git checkout preprod  # ou main/develop selon le projet
git pull origin preprod

# 2. Créer la nouvelle branche
git checkout -b <type>/<description>

# 3. Pousser avec tracking
git push -u origin <type>/<description>
```

## Pull Requests

### Format Summary + Test Plan

```markdown
## Summary
- <changement principal 1>
- <changement principal 2>
- <changement principal 3>

## Test plan
- [ ] <test à effectuer 1>
- [ ] <test à effectuer 2>
- [ ] <test à effectuer 3>
```

### Workflow PR

```bash
# 1. Vérifier l'état de la branche
git status
git diff preprod...HEAD
git log preprod..HEAD --oneline

# 2. Pousser les changements
git push -u origin <branche>

# 3. Créer la PR avec gh
gh pr create --title "<type>(<scope>): <description>" --body "$(cat <<'EOF'
## Summary
- <bullet points des changements>

## Test plan
- [ ] <checklist de tests>
EOF
)"
```

### Titre de PR

Le titre suit le même format que les commits conventionnels :
```
<type>(<scope>): <description courte>
```

## Changelog

### Génération automatique

Analyser les commits depuis le dernier tag pour générer un changelog :

```bash
# Récupérer les commits depuis le dernier tag
git log $(git describe --tags --abbrev=0)..HEAD --oneline
```

### Format changelog

```markdown
## [Version] - YYYY-MM-DD

### Added
- feat(scope): description

### Fixed
- fix(scope): description

### Changed
- refactor(scope): description

### Documentation
- docs(scope): description
```

### Mapping type → section

| Type commit | Section changelog |
|-------------|-------------------|
| `feat` | Added |
| `fix` | Fixed |
| `refactor`, `style`, `chore` | Changed |
| `docs` | Documentation |
| `test` | (non inclus par défaut) |

## Validation Pre-commit

Avant chaque commit, vérifier :

1. **Format du message**
   - Type valide (feat, fix, docs, style, refactor, test, chore)
   - Scope présent entre parenthèses
   - Description après les deux-points
   - Longueur ≤ 72 caractères

2. **Contenu**
   - Fichiers sensibles exclus (.env, credentials, secrets)
   - Documentation IA exclue (CLAUDE.md, AGENTS.md, openspec/, .claude/, docs/)
   - Pas de console.log/var_dump de debug
   - Tests passent (si applicable)

3. **Atomicité**
   - Un commit par scope
   - Pas de commit mélangeant plusieurs domaines fonctionnels

3. **Exemple de validation**
   ```
   ✓ feat(api): ajouter endpoint utilisateurs
   ✗ feat: ajouter endpoint (scope manquant)
   ✗ ajout endpoint (type manquant)
   ✗ feat(api): Ajouter endpoint. (majuscule + point)
   ```

## Exemples complets

### Nouvelle fonctionnalité (commits scope par scope)

```bash
# Créer la branche
git checkout -b feature/export-csv

# Développer...

# Commit 1 : le service
git add src/Export/Service/ExportService.php
git commit -m "feat(export): ajouter service ExportService"

# Commit 2 : le controller/endpoint
git add src/Export/Controller/ExportController.php
git commit -m "feat(api): ajouter endpoint GET /api/users/export"

# Commit 3 : les tests
git add tests/Export/
git commit -m "test(export): ajouter tests unitaires export CSV"

# PR
gh pr create --title "feat(export): ajouter export CSV des utilisateurs" --body "$(cat <<'EOF'
## Summary
- Ajout d'un service ExportService pour générer des CSV
- Nouvel endpoint GET /api/users/export
- Support des filtres de recherche existants

## Test plan
- [ ] Tester l'export sans filtres
- [ ] Tester l'export avec filtres
- [ ] Vérifier le format CSV généré
- [ ] Tester avec un grand volume de données
EOF
)"
```

### Correction de bug

```bash
# Créer la branche
git checkout -b fix/validation-email

# Corriger...

# Commit (un seul scope = un seul commit)
git add src/User/Validator/EmailValidator.php
git commit -m "fix(user): corriger validation emails avec caractères spéciaux"

# Si tests ajoutés, commit séparé
git add tests/User/Validator/EmailValidatorTest.php
git commit -m "test(user): ajouter tests validation email"

# PR
gh pr create --title "fix(user): corriger validation email" --body "$(cat <<'EOF'
## Summary
- Correction du regex de validation email
- Support des caractères + et . dans la partie locale

## Test plan
- [ ] Tester email standard (user@domain.com)
- [ ] Tester email avec + (user+tag@domain.com)
- [ ] Tester email avec . (user.name@domain.com)
- [ ] Vérifier rejet des emails invalides
EOF
)"
```