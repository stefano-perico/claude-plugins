# Aikido Security — API REST Endpoints

Base URL : `https://app.aikido.dev/api/public/v1`
Régions alternatives : `https://app.us.aikido.dev/api/public/v1` (US), `https://app.me.aikido.dev/api/public/v1` (ME)

## Authentification

### Obtenir un token

```
POST https://app.aikido.dev/api/oauth/token
```

**Headers :**

```
Authorization: Basic base64(AIKIDO_CLIENT_ID:AIKIDO_CLIENT_SECRET)
Content-Type: application/x-www-form-urlencoded
```

**Body :**

```
grant_type=client_credentials
```

**Réponse (200) :**

```json
{
  "access_token": "eyJhbGciOi...",
  "expires_in": 3600,
  "token_type": "bearer"
}
```

**Erreurs :**
- `400` — `invalid_client` (mauvais credentials), `unsupported_grant_type`, `invalid_request`

> Utiliser ensuite le header `Authorization: Bearer {access_token}` pour tous les appels API.

---

## Lister les repositories

```
GET /repositories/code
```

**Scope OAuth :** `repositories:read`

**Paramètres query :**

| Paramètre | Type | Description |
|-----------|------|-------------|
| `page` | integer | Page (commence à 0) |
| `per_page` | integer | Résultats par page (10-20, défaut 20) |

**Réponse :**

```json
[
  {
    "id": 12345,
    "name": "my-app",
    "provider": "github",
    "environment": "production",
    "external_id": "123456789",
    "last_scanned": 1700000000,
    "status": "active"
  }
]
```

---

## Lister les issues ouvertes (groupes)

```
GET /open-issue-groups
```

**Scope OAuth :** `issues:read`

**Paramètres query :**

| Paramètre | Type | Description |
|-----------|------|-------------|
| `page` | integer | Page (commence à 0, défaut 0) |
| `per_page` | integer | Résultats par page (10-20, défaut 20) |
| `filter_code_repo_id` | integer | Filtrer par ID de repository |
| `filter_external_code_repo_id` | string | ID du repo chez le provider |
| `filter_code_repo_name` | string | Filtrer par nom de repository |
| `filter_container_repo_id` | integer | Filtrer par container repository |
| `filter_team_id` | integer | Filtrer par équipe |
| `filter_issue_type` | string | Type : `open_source`, `leaked_secret`, `cloud`, `sast`, `iac`, `docker_container`, `cloud_instance`, `surface_monitoring`, `malware`, `eol`, `mobile`, `scm_security`, `ai_pentest`, `license` |

**Réponse :**

```json
[
  {
    "id": 42,
    "title": "SQL Injection in UserController",
    "description": "Unsanitized user input...",
    "type": "sast",
    "severity_score": 85,
    "severity": "critical",
    "group_status": "new",
    "time_to_fix_minutes": 30,
    "locations": [...],
    "how_to_fix": "Use parameterized queries...",
    "related_cve_ids": ["CVE-2024-1234"]
  }
]
```

---

## Détail d'un groupe d'issues

```
GET /issues/groups/{issue_group_id}
```

**Scope OAuth :** `issues:read`

**Path parameter :** `issue_group_id` (integer, requis)

**Réponse :** même structure qu'un élément de `/open-issue-groups` avec les champs `id`, `title`, `description`, `type`, `severity_score`, `severity`, `group_status`, `time_to_fix_minutes`, `locations`, `how_to_fix`, `related_cve_ids`.

---

## Exporter les issues (détail)

```
GET /issues/export
```

**Scope OAuth :** `issues:read`

**Paramètres query :**

| Paramètre | Type | Description |
|-----------|------|-------------|
| `format` | string | `json` ou `csv` (défaut `json`) |
| `filter_status` | string | `all`, `open`, `ignored`, `snoozed`, `closed` (défaut `all`) |
| `filter_issue_group_id` | integer | Filtrer par groupe d'issue |
| `filter_code_repo_id` | integer | Filtrer par repository |
| `filter_code_repo_name` | string | Filtrer par nom de repo |
| `filter_container_repo_id` | integer | Filtrer par container |
| `filter_container_repo_name` | string | Filtrer par nom de container |
| `filter_team_id` | integer | Filtrer par équipe |
| `filter_issue_type` | string | Type (mêmes valeurs que `/open-issue-groups`) |
| `filter_severities` | string | Séparées par virgule : `critical`, `high`, `medium`, `low` |
| `filter_language` | string | Langage de programmation |

**Réponse (JSON) :**

```json
[
  {
    "id": 1001,
    "group_id": 42,
    "type": "sast",
    "rule": "sql-injection",
    "rule_id": "SAST-001",
    "severity_score": 85,
    "severity": "critical",
    "status": "open",
    "affected_package": null,
    "affected_file": "src/Controller/UserController.php",
    "cve_id": null,
    "first_detected_at": 1700000000,
    "code_repo_id": 12345,
    "code_repo_name": "my-app",
    "start_line": 42,
    "end_line": 45,
    "installed_version": null,
    "patched_versions": [],
    "programming_language": "php",
    "cwe_classes": ["CWE-89"]
  }
]
```

---

## Déclencher un scan

```
POST /repositories/code/{code_repo_id}/scan
```

**Scope OAuth :** `repositories:write`

**Path parameter :** `code_repo_id` (integer, requis)

**Réponse (200) :**

```json
{
  "status": "ok"
}
```
