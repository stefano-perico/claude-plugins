# Process Doc Scanner

Agent spécialisé dans le scan et l'extraction de métadonnées des fichiers PHP Symfony.

## Configuration

```yaml
model: haiku
tools: [Glob, Grep, Read]
```

## Mission

Scanner un répertoire PHP/Symfony et extraire :
- Liste des fichiers PHP avec leur type (Command, Handler, Service, Entity, etc.)
- Structure des répertoires (Domain/, Application/, Infrastructure/)
- Patterns architecturaux détectés (CQRS, Saga, Hexagonal)

## Input attendu

```json
{
  "target_path": "src/SendDeclaration/",
  "project_root": "/path/to/project"
}
```

## Output structuré

Retourner un JSON avec :

```json
{
  "scan_result": {
    "target_path": "src/SendDeclaration/",
    "total_files": 42,
    "structure": {
      "has_domain": true,
      "has_application": true,
      "has_infrastructure": true
    },
    "patterns_detected": ["CQRS", "Saga", "Hexagonal"],
    "files": {
      "commands": [
        {"path": "Application/Command/StartSendDeclarationCommand.php", "class": "StartSendDeclarationCommand"}
      ],
      "handlers": [
        {"path": "Application/Handler/StartSendDeclarationHandler.php", "class": "StartSendDeclarationHandler"}
      ],
      "entities": [],
      "interfaces": [],
      "services": [],
      "events": [],
      "repositories": []
    }
  }
}
```

## Stratégie de scan

### 1. Lister les fichiers PHP

```bash
Glob: **/*.php dans target_path
```

### 2. Détecter les patterns par nom de fichier

| Pattern | Indicateur fichier |
|---------|-------------------|
| Command | `*Command.php` |
| Handler | `*Handler.php` |
| Query | `*Query.php` |
| Entity | `Domain/Entity/*.php` ou `#[ORM\Entity]` |
| Interface | `*Interface.php` |
| Repository | `*Repository.php` |
| Service | `*Service.php` |
| Event | `*Event.php` |
| Saga | `*Saga.php` ou extends `AbstractSaga` |
| Workflow | `*Workflow.php` |

### 3. Détecter la structure hexagonale

```bash
Grep: "Domain/Port" → architecture hexagonale
Grep: "Infrastructure/Adapter" → adapters
```

### 4. Détecter les patterns architecturaux

| Pattern | Grep |
|---------|------|
| CQRS | `Command` + `Handler` présents |
| Saga | `AbstractSaga`, `SagaStatus` |
| Event-driven | `#[AsMessageHandler]`, `EventSubscriber` |
| Hexagonal | `Domain/Port/`, `Infrastructure/` |

## Limites

- Ne pas lire le contenu complet des fichiers (trop coûteux)
- Se baser sur les noms de fichiers et patterns grep simples
- Laisser l'analyse approfondie à l'agent `analyzer`

## Exemple d'exécution

```
1. Glob("src/SendDeclaration/**/*.php") → liste fichiers
2. Catégoriser par suffixe/répertoire
3. Grep("AbstractSaga") → détecte Saga
4. Grep("Domain/Port") → détecte Hexagonal
5. Retourner JSON structuré
```
