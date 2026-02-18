# Aikido MCP Server — Installation et outils

## Prérequis

- Node.js (pour `npx`)
- Un token API Aikido (Personal Access Token)

## Générer le token

1. Aller sur [Aikido](https://app.aikido.dev) → **Settings** → **Integrations** → **IDE** → **MCP**
2. Cliquer sur **Generate Token**
3. Copier le token généré

## Installation pour Claude Code

```bash
claude mcp add aikido -- npx -y @aikidosec/mcp
```

Puis configurer la variable d'environnement :

```bash
export AIKIDO_API_KEY="votre_token_ici"
```

Ou ajouter dans le fichier de configuration MCP de Claude Code (`.claude/settings.json`) :

```json
{
  "mcpServers": {
    "aikido": {
      "command": "npx",
      "args": ["-y", "@aikidosec/mcp"],
      "env": {
        "AIKIDO_API_KEY": "votre_token_ici"
      }
    }
  }
}
```

## Outils disponibles

### `aikido_full_scan`

Scan combiné SAST + Secrets sur les fichiers fournis. C'est le scan le plus complet.

### `aikido_sast_scan`

Scan SAST (Static Application Security Testing) local uniquement. Détecte les vulnérabilités dans le code source (injections SQL, XSS, etc.).

### `aikido_secrets_scan`

Scan de secrets uniquement. Détecte les secrets codés en dur (clés API, mots de passe, tokens).

## Utilisation dans un workflow

Après avoir appliqué un fix, lancer un scan pour vérifier la correction :

1. `aikido_full_scan` pour un scan complet
2. `aikido_sast_scan` pour vérifier un fix SAST spécifique
3. `aikido_secrets_scan` pour vérifier qu'un secret a bien été supprimé
