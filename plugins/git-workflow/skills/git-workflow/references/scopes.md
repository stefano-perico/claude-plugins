# Scopes recommandés

Liste des scopes couramment utilisés, organisés par domaine.

## Backend / API

| Scope | Description |
|-------|-------------|
| `api` | Endpoints REST/GraphQL |
| `auth` | Authentification, autorisation |
| `db` | Base de données, migrations |
| `cache` | Cache Redis, APCu |
| `queue` | Files de messages, workers |
| `config` | Configuration applicative |
| `security` | Sécurité, CORS, CSRF |

## Frontend

| Scope | Description |
|-------|-------------|
| `ui` | Composants d'interface |
| `form` | Formulaires, validation |
| `router` | Navigation, routes |
| `store` | State management |
| `i18n` | Internationalisation |

## Métier (Inter-Invest)

| Scope | Description |
|-------|-------------|
| `defisc` | Défiscalisation |
| `girardin` | Produit Girardin |
| `scpi` | Produit SCPI |
| `per` | Plan Épargne Retraite |
| `assurance-vie` | Assurance vie |
| `client` | Gestion clients |
| `distributeur` | Gestion distributeurs |
| `document` | Génération documents |
| `signature` | Signature électronique |
| `paiement` | Paiements, prélèvements |
| `ebics` | Intégration EBICS/SEPA |
| `spv` | Sociétés de portage |

## Infrastructure

| Scope | Description |
|-------|-------------|
| `docker` | Configuration Docker |
| `k8s` | Kubernetes, Helm |
| `ci` | CI/CD pipelines |
| `deps` | Dépendances, packages |
| `env` | Variables d'environnement |

## Documentation

| Scope | Description |
|-------|-------------|
| `readme` | Fichier README |
| `changelog` | CHANGELOG |
| `api-docs` | Documentation API |
| `comments` | Commentaires de code |

## Bonnes pratiques

1. **Cohérence** - Utiliser le même scope pour le même domaine
2. **Spécificité** - Préférer un scope précis à un scope générique
3. **Minuscules** - Toujours en minuscules, sans espaces
4. **Kebab-case** - Pour les scopes composés: `api-client`, `user-profile`
