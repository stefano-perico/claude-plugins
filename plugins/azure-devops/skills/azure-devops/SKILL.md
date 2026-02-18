---
name: azure-devops
description: Gestion des work items Azure DevOps (Epics, User Stories, Tasks) via la CLI az. Utiliser ce skill pour lister mes items assignés, créer de nouveaux work items, modifier états/titres/assignations, déplacer entre sprints, et naviguer entre projets d'une même organisation. Déclencher quand l'utilisateur demande ses tasks, US, epics, ou veut interagir avec Azure DevOps.
---

# Azure DevOps Work Items

Gestion des Epics, User Stories et Tasks via `az boards`.

## Prérequis

Vérifier que l'utilisateur a configuré ses defaults :
```bash
az devops configure --defaults organization=https://dev.azure.com/ORG project=PROJET
```

## Commandes rapides

### Lister mes items

```bash
# Mes tasks actives
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.WorkItemType] = 'Task' AND [System.AssignedTo] = '@Me' AND [System.State] <> 'Closed'" -o table

# Mes User Stories
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.WorkItemType] = 'User Story' AND [System.AssignedTo] = '@Me'" -o table

# Tout ce qui m'est assigné
az boards query --wiql "SELECT [System.Id], [System.Title], [System.WorkItemType], [System.State] FROM WorkItems WHERE [System.AssignedTo] = '@Me' ORDER BY [System.ChangedDate] DESC" -o table

# Sprint courant
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.AssignedTo] = '@Me' AND [System.IterationPath] = @CurrentIteration" -o table
```

### Voir un item spécifique

```bash
az boards work-item show --id <ID> -o table
```

### Créer un work item

```bash
# Task
az boards work-item create --type Task --title "Titre" --assigned-to "email@domain.com"

# User Story
az boards work-item create --type "User Story" --title "Titre" --assigned-to "email@domain.com"

# Epic
az boards work-item create --type Epic --title "Titre"

# Sur un projet spécifique
az boards work-item create --type Task --title "Titre" --project "NomProjet"
```

### Modifier un work item

```bash
# Changer l'état
az boards work-item update --id <ID> --state "In Progress"
az boards work-item update --id <ID> --state "Done"

# Modifier titre
az boards work-item update --id <ID> --title "Nouveau titre"

# Réassigner
az boards work-item update --id <ID> --assigned-to "autre@domain.com"

# Changer de sprint
az boards work-item update --id <ID> --iteration "Projet\\Sprint 2"
```

### Lier des items

```bash
# Lier une task à une US (parent)
az boards work-item relation add --id <TASK_ID> --relation-type parent --target-id <US_ID>
```

### Naviguer entre projets

```bash
# Lister les projets
az devops project list -o table

# Requête sur un autre projet
az boards query --project "AutreProjet" --wiql "SELECT ... FROM WorkItems WHERE ..." -o table
```

## États disponibles

| Type | États |
|------|-------|
| Epic/US | New, Active, Resolved, Closed |
| Task | New, Active, Closed |

## Référence complète

Pour les commandes avancées et options détaillées, voir [references/az-boards-commands.md](references/az-boards-commands.md).
