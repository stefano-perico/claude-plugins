# Stefano Plugins

Skills et agents Claude Code pour Symfony, Git, Azure DevOps et developpement PHP.

## Plugins

| Plugin | Description | Category |
|:-------|:------------|:---------|
| **symfony-dev-team** | Team d'agents pour developper des features Symfony en architecture hexagonale + CQRS | Development |
| **symfony-quality** | Audit SOLID et corrections automatisees pour code Symfony | Development |
| **symfony-process-doc** | Documentation technique de processus Symfony avec diagrammes Mermaid | Productivity |
| **git-workflow** | Workflow Git avec Conventional Commits, branches et Pull Requests | Productivity |
| **azure-devops** | Gestion des work items Azure DevOps via CLI | Productivity |
| **skill-creator** | Guide complet pour creer, tester et distribuer des skills Claude Code | Development |

## Installation

### 1. Ajouter la marketplace

```
/plugin marketplace add stefanoperico/claude-plugins
```

### 2. Installer un plugin

```
/plugin install symfony-dev-team@stefano-plugins
/plugin install symfony-quality@stefano-plugins
/plugin install symfony-process-doc@stefano-plugins
/plugin install git-workflow@stefano-plugins
/plugin install azure-devops@stefano-plugins
/plugin install skill-creator@stefano-plugins
```

### 3. Utiliser les skills

```
/launch-dev-team          # Lance une team de dev Symfony
/launch-audit-team        # Lance un audit SOLID avec corrections
/symfony-process-doc      # Genere la documentation technique
/git-workflow             # Gestion Git avec Conventional Commits
/azure-devops             # Gestion des work items Azure DevOps
/skill-creator            # Guide de creation de skills
```

## Prerequis

| Plugin | Prerequis |
|:-------|:----------|
| symfony-dev-team | PHP 8.3+, Symfony 7+, Composer |
| symfony-quality | PHP 8.3+, Symfony 7+, PHPStan, PHPUnit |
| symfony-process-doc | PHP 8.3+, Symfony 7+ |
| git-workflow | Git |
| azure-devops | Azure CLI (`az`) avec extension `azure-devops` |
| skill-creator | Claude Code |

## Structure

Chaque plugin suit la structure standard :

```
plugins/{plugin-name}/
  .claude-plugin/
    plugin.json            # Manifeste du plugin
  skills/
    {skill-name}/
      SKILL.md             # Instructions du skill
      references/           # Fichiers de reference
      scripts/              # Scripts optionnels
  agents/                   # Agents optionnels
    {agent-name}.md
```

## Versioning

- Chaque `plugin.json` contient sa propre version (semver)
- Tags git pour les releases : `v1.0.0`, `v1.1.0`, etc.
- Mise a jour : `/plugin marketplace update`

## License

MIT
