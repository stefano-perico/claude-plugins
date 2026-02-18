# Aikido MCP Server — Installation et outils

> Source : https://help.aikido.dev/ide-plugins/aikido-mcp/anthropic-claude-code-mcp

## Prérequis

- Node.js (pour `npx`)
- Un compte Aikido avec accès aux settings IDE

## Étape 1 : Générer le token API

1. Aller sur [Aikido](https://app.aikido.dev) → **Settings** → **Integrations** → **IDE** → **MCP**
2. Cliquer sur **Generate Token**
3. Copier le token généré (Personal Access Token)

## Étape 2 : Installer le MCP Server

Lancer cette commande (remplacer `YOUR_TOKEN` par le token copié) :

```bash
claude mcp add --transport stdio aikido \
  --env AIKIDO_API_KEY=YOUR_TOKEN \
  -- npx -y @aikidosec/mcp
```

Cette commande enregistre le serveur MCP dans la config locale Claude Code (`~/.claude.json`).

## Étape 3 : Installer la skill rule Aikido

Créer le dossier skills si nécessaire, puis télécharger la rule :

```bash
mkdir -p ~/.claude/skills/

curl -fsSL "https://gist.githubusercontent.com/kidk/aa48cad6db80ba4a38493016aae67712/raw/3644397b7df43423e3da06434491b40bbb79dd47/aikido-rule.txt" \
  -o ~/.claude/skills/aikido-rule.txt
```

## Étape 4 : Activer

Relancer Claude Code (quitter et rouvrir) pour que le MCP soit chargé.

Vérifier avec `/mcp` que le serveur `aikido` apparaît dans la liste.

## Outils disponibles

Le MCP expose 3 outils de scan local :

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
