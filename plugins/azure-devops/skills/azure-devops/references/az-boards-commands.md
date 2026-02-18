# Azure DevOps CLI - Commandes az boards

## Configuration requise

```bash
# Configurer les defaults (une seule fois)
az devops configure --defaults organization=https://dev.azure.com/MON_ORG project=MON_PROJET
```

## Lister les work items

### Par requête WIQL
```bash
# Mes tasks assignées
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.WorkItemType] = 'Task' AND [System.AssignedTo] = '@Me'" -o table

# Mes User Stories
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.WorkItemType] = 'User Story' AND [System.AssignedTo] = '@Me'" -o table

# Mes items actifs (non fermés)
az boards query --wiql "SELECT [System.Id], [System.Title], [System.WorkItemType], [System.State] FROM WorkItems WHERE [System.AssignedTo] = '@Me' AND [System.State] <> 'Closed' AND [System.State] <> 'Done'" -o table

# Items du sprint courant
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.AssignedTo] = '@Me' AND [System.IterationPath] = @CurrentIteration" -o table
```

### Afficher un work item spécifique
```bash
az boards work-item show --id <ID> -o table
az boards work-item show --id <ID> -o json  # détails complets
```

## Créer des work items

### Task
```bash
az boards work-item create --type Task --title "Ma nouvelle task" --assigned-to "email@domain.com" --description "Description de la task"
```

### User Story
```bash
az boards work-item create --type "User Story" --title "Ma US" --assigned-to "email@domain.com"
```

### Epic
```bash
az boards work-item create --type Epic --title "Mon Epic" --assigned-to "email@domain.com"
```

### Avec projet spécifique
```bash
az boards work-item create --type Task --title "Task" --project "NomDuProjet"
```

## Modifier des work items

### Changer l'état
```bash
az boards work-item update --id <ID> --state "In Progress"
az boards work-item update --id <ID> --state "Done"
az boards work-item update --id <ID> --state "Active"
```

### Modifier le titre
```bash
az boards work-item update --id <ID> --title "Nouveau titre"
```

### Réassigner
```bash
az boards work-item update --id <ID> --assigned-to "autre@domain.com"
```

### Modifier plusieurs champs
```bash
az boards work-item update --id <ID> --fields "System.Title=Nouveau titre" "System.State=Active" "System.AssignedTo=email@domain.com"
```

### Ajouter à une itération/sprint
```bash
az boards work-item update --id <ID> --iteration "Projet\\Sprint 1"
```

### Lier à un parent
```bash
az boards work-item relation add --id <CHILD_ID> --relation-type parent --target-id <PARENT_ID>
```

## Lister les projets

```bash
az devops project list -o table
```

## Lister les itérations

```bash
az boards iteration project list -o table
az boards iteration team list --team "Team Name" -o table
```

## États courants (Process Agile/Scrum)

| Type | États disponibles |
|------|-------------------|
| Epic | New, Active, Resolved, Closed |
| User Story | New, Active, Resolved, Closed |
| Task | New, Active, Closed |
| Bug | New, Active, Resolved, Closed |

## Formats de sortie

- `-o table` : Tableau lisible
- `-o json` : JSON complet
- `-o tsv` : Valeurs séparées par tabulation
- `-o yaml` : Format YAML
