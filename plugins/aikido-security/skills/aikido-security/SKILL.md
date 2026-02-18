---
name: aikido-security
description: |
  Scan de sécurité et résolution d'issues via Aikido Security (API REST + MCP).
  Utiliser ce skill pour : (1) Lister et analyser les issues de sécurité Aikido d'un repository,
  (2) Récupérer le détail d'une vulnérabilité (description, how_to_fix, CVEs, fichiers affectés),
  (3) Résoudre des vulnérabilités (open_source, sast, leaked_secret, iac),
  (4) Scanner le code localement via le MCP server @aikidosec/mcp,
  (5) Vérifier qu'une correction est effective après application.
  Déclencher quand l'utilisateur demande : /aikido, issues aikido, vulnérabilités aikido,
  "issues aikido", "failles aikido", "résoudre CVE aikido", ou toute interaction avec Aikido Security.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Glob
  - Grep
---

# Aikido Security

Skill pour scanner, analyser et résoudre les issues de sécurité via Aikido Security.

## 1. Prérequis

Vérifier que les variables d'environnement sont configurées :

**API REST (obligatoire) :**
- `AIKIDO_CLIENT_ID` — Client ID OAuth2
- `AIKIDO_CLIENT_SECRET` — Client Secret OAuth2

**MCP Server (optionnel, pour scan local) :**
- `AIKIDO_API_KEY` — Token généré dans Settings → Integrations → IDE → MCP

### Vérification

```bash
# Vérifier les credentials API REST
test -n "$AIKIDO_CLIENT_ID" && test -n "$AIKIDO_CLIENT_SECRET" && echo "API REST OK" || echo "API REST: AIKIDO_CLIENT_ID et AIKIDO_CLIENT_SECRET requis"

# Vérifier le MCP (optionnel)
test -n "$AIKIDO_API_KEY" && echo "MCP OK" || echo "MCP non configuré (optionnel)"
```

Si les variables sont absentes, guider l'utilisateur :

1. **API REST** : Aller sur https://app.aikido.dev → Settings → Integrations → scroll en bas → **Public REST API** → Manage → **Add Client** (nécessite rôle workspace admin)
2. **MCP** : Voir `references/mcp-setup.md` pour l'installation

## 2. Authentification API

Obtenir un bearer token avant tout appel API :

```bash
TOKEN=$(curl -s -X POST "https://app.aikido.dev/api/oauth/token" \
  -H "Authorization: Basic $(echo -n "${AIKIDO_CLIENT_ID}:${AIKIDO_CLIENT_SECRET}" | base64)" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" | jq -r '.access_token')
```

Le token est valide 1h (`expires_in: 3600`). Le stocker pour réutilisation dans la session.

Pour tous les appels suivants, utiliser :

```
Authorization: Bearer $TOKEN
```

### Gestion d'erreur

Après chaque appel API, vérifier le code HTTP :

| Code | Cause | Action |
|------|-------|--------|
| `401` | Token expiré ou invalide | Régénérer le token (étape 2) |
| `403` | Scope OAuth insuffisant | Vérifier les permissions du client OAuth2 sur Aikido |
| `404` | Ressource introuvable | Vérifier l'ID repo/issue utilisé |
| `429` | Rate limit atteint | Attendre 60s puis réessayer |

Toujours parser la réponse JSON et vérifier la présence de `access_token` ou des données attendues avant de continuer.

## 3. Input utilisateur

Accepter optionnellement une URL Aikido pour cibler directement un repo ou une issue :

| Format URL | Action |
|------------|--------|
| `https://app.aikido.dev/repositories/{id}` | Utiliser `filter_code_repo_id={id}` pour lister les issues |
| `https://app.aikido.dev/issues/group/{id}` | Aller au détail du groupe d'issue via `GET /issues/groups/{id}` |
| Pas d'URL | Demander le nom du repo ou lister tous les repos via `GET /repositories/code` |

### Parser l'URL

- Extraire l'ID du segment de path : `/repositories/(\d+)` ou `/issues/group/(\d+)`
- Si URL repo → enchaîner avec l'étape 4 (lister les issues filtrées)
- Si URL issue group → enchaîner avec l'étape 5 (détailler l'issue)

## 4. Lister les issues

```bash
curl -s "https://app.aikido.dev/api/public/v1/open-issue-groups?filter_code_repo_id={REPO_ID}&per_page=20&page=0" \
  -H "Authorization: Bearer $TOKEN"
```

### Filtres disponibles

| Paramètre | Usage |
|-----------|-------|
| `filter_code_repo_id` | Par ID de repo (depuis URL ou étape 3) |
| `filter_code_repo_name` | Par nom de repo (si pas d'ID) |
| `filter_issue_type` | `sast`, `open_source`, `leaked_secret`, `iac`, `eol`, `malware`, `license`... |
| `page` / `per_page` | Pagination (max 20 par page, page commence à 0) |

### Affichage

Présenter les résultats sous forme de tableau :

| ID | Titre | Sévérité | Type | Fix estimé |
|----|-------|----------|------|------------|

Trier par sévérité (critical > high > medium > low).

Si l'utilisateur veut traiter une issue, passer à l'étape 5.

## 5. Détailler une issue

Deux endpoints selon le besoin :

### Détail du groupe

```bash
curl -s "https://app.aikido.dev/api/public/v1/issues/groups/{ISSUE_GROUP_ID}" \
  -H "Authorization: Bearer $TOKEN"
```

Afficher : `title`, `description`, `how_to_fix`, `related_cve_ids`, `severity`, `time_to_fix_minutes`.

### Export des issues individuelles du groupe

```bash
curl -s "https://app.aikido.dev/api/public/v1/issues/export?filter_issue_group_id={ISSUE_GROUP_ID}&filter_status=open&format=json" \
  -H "Authorization: Bearer $TOKEN"
```

Afficher pour chaque issue : `affected_file`, `start_line`-`end_line`, `affected_package`, `installed_version`, `patched_versions`, `cwe_classes`.

## 6. Résoudre

Appliquer le fix selon le type d'issue :

### Par type d'issue

| Type | Action |
|------|--------|
| `open_source` | Identifier le package et `patched_versions` depuis l'export. Mettre à jour la dépendance avec le gestionnaire du projet (composer, npm, pip). Vérifier que les tests passent. |
| `sast` | Lire `affected_file` aux lignes `start_line`-`end_line`. Appliquer la correction décrite dans `how_to_fix`. |
| `leaked_secret` | Supprimer le secret du code, le remplacer par une variable d'env, l'ajouter dans `.env.example` (sans valeur). Informer l'utilisateur de faire tourner (rotate) le secret. |
| `iac` | Lire le fichier de config affecté. Appliquer la correction selon `how_to_fix`. |

## 7. Valider la correction

### Avec MCP (si configuré)

Si le MCP server `@aikidosec/mcp` est disponible, lancer un scan local pour vérifier :

- Après fix SAST → `aikido_sast_scan` sur le fichier corrigé
- Après fix secret → `aikido_secrets_scan` sur le fichier corrigé
- Pour une validation complète → `aikido_full_scan`

### Sans MCP

Déclencher un scan côté Aikido via l'API :

```bash
curl -s -X POST "https://app.aikido.dev/api/public/v1/repositories/code/{REPO_ID}/scan" \
  -H "Authorization: Bearer $TOKEN"
```

Puis vérifier sur l'interface Aikido que l'issue est résolue après le scan.

Voir `references/api-endpoints.md` pour les paramètres complets et formats de réponse.
Voir `references/mcp-setup.md` pour l'installation du MCP server.
