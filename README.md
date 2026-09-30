# Annuaire des gardes et astreintes

Annuaire du personnel de garde de l'Institut Mutualiste Montsouris (IMM) : recherche du personnel, numéros de garde et numéros d'urgence. La consultation est publique ; l'administration (création, modification, suppression, journal des actions) est réservée aux comptes Active Directory.

Le projet se compose de deux dépôts :

| Dépôt | Contenu | Technologies |
|---|---|---|
| [API-REST-annuaire](https://github.com/Otto-Cyril/API-REST-annuaire) | API JSON, authentification LDAP + JWT, journal d'audit | Symfony 7.4, Doctrine, SQL Server |
| [Frontend-annuaire](https://github.com/Otto-Cyril/Frontend-annuaire) | Interface web (annuaire, fiches, administration) | Vue 3, Vue Router, Pinia, Vite |

## Démarrage rapide

1. **API** : configurer et installer selon la partie [API REST](#api-rest) ci-dessous, puis :

   ```bash
   php -S 127.0.0.1:8000 -t public      # ou : symfony serve
   ```

2. **Frontend** (Node.js requis), depuis le dépôt du frontend :

   ```bash
   npm install
   npm run dev
   ```

   L'application est servie sur http://localhost:5173/. En développement, Vite relaie `/api` vers `http://127.0.0.1:8000` (`vite.config.js`) : l'API doit donc tourner sur ce port.

## Tests

```bash
php bin/phpunit          # API (base de test requise, voir « Tests de l'API »)
npm test                 # frontend (Vitest)
```

## Avant une mise en production

- Renseigner `LDAP_ADMIN_GROUP_DN` : sinon tout compte AD valide obtient les droits d'écriture.
- Passer LDAP en `ssl`/`tls` : en `none`, les mots de passe circulent en clair.
- Définir `APP_ENV=prod`, un `APP_SECRET` et un `JWT_PASSPHRASE` propres, et `CORS_ALLOW_ORIGIN` pour l'URL du front.
- Utiliser un compte SQL Server à privilèges limités (pas `sa`) et retirer `TrustServerCertificate` si le certificat est valide.
- Frontend : `npm run build` produit `dist/`, à servir comme site statique avec `/api` relayé vers l'API (voir « Déploiement du frontend »).

---

# API REST

API REST (Symfony 7.4) qui expose l'annuaire du personnel de garde, de ses numéros de garde et des numéros d'urgence.
La lecture est publique ; la création, la modification et la suppression exigent un JWT obtenu en se connectant avec
un compte Active Directory. Chaque écriture est journalisée (table `trace`).

## Prérequis

- PHP ≥ 8.2 avec les extensions `ldap`, `openssl`, `ctype`, `iconv`, `pdo_sqlsrv` et `sqlsrv`
- [Composer](https://getcomposer.org/)
- SQL Server (2022 en dev via Docker, voir plus bas) et le pilote ODBC « Microsoft ODBC Driver 18 »
- Accès à l'Active Directory pour la connexion (`immdom.local`)

## Installation

```bash
composer install
```

### 1. Configuration (`.env.local`)

Le fichier `.env` est ignoré par git : les valeurs ci-dessous sont à mettre dans un fichier `.env.local`
(ignoré aussi, ne jamais le commiter).

| Variable | Rôle | Exemple |
|---|---|---|
| `APP_ENV` | Environnement | `dev` |
| `APP_SECRET` | Secret Symfony (chaîne aléatoire) | `4adf91…` |
| `DATABASE_URL` | Connexion SQL Server | `pdo-sqlsrv://utilisateur:motdepasse@127.0.0.1:1433/ANNUAIRE_IMM_DEV?charset=utf8&TrustServerCertificate=true` |
| `JWT_SECRET_KEY` / `JWT_PUBLIC_KEY` | Chemins des clés JWT | `%kernel.project_dir%/config/jwt/private.pem` / `public.pem` |
| `JWT_PASSPHRASE` | Phrase secrète de la clé privée | (aléatoire) |
| `CORS_ALLOW_ORIGIN` | Origines autorisées (regex) | `^https?://(localhost\|127\.0\.0\.1)(:[0-9]+)?$` |
| `LDAP_HOST` / `LDAP_PORT` / `LDAP_ENCRYPTION` | Serveur AD | `immdom.local` / `389` / `none` (`ssl` ou `tls` en prod) |
| `LDAP_BASE_DN` | Où chercher les utilisateurs | `OU=Comptes,OU=IMM,DC=immdom,DC=local` |
| `LDAP_SEARCH_DN` / `LDAP_SEARCH_PASSWORD` | Compte technique de recherche | |
| `LDAP_USER_QUERY` | Filtre de recherche de l'utilisateur | `(sAMAccountName={username})` |
| `LDAP_ADMIN_GROUP_DN` | DN du groupe AD autorisé à écrire. **Vide = tout compte AD valide est admin** | (vide pour l'instant) |

### 2. Clés JWT

```bash
php bin/console lexik:jwt:generate-keypair
```

Sous Windows, si OpenSSL ne trouve pas sa configuration, définir `OPENSSL_CONF` (ex. `C:/php/extras/ssl/openssl.cnf`).

### 3. Base de données

Une instance SQL Server de développement est fournie :

```bash
docker compose --env-file .env.local up -d database   # SQL Server sur 127.0.0.1:1433
php bin/console doctrine:database:create
php bin/console doctrine:migrations:migrate
```

**Le mot de passe `sa` du conteneur doit être le même que celui de `DATABASE_URL`.** Ajouter dans `.env.local` :

```
MSSQL_SA_PASSWORD=le_meme_mot_de_passe_que_dans_DATABASE_URL
```

Docker Compose ne lit pas `.env.local` tout seul : d'où l'option `--env-file .env.local` (ou définir la variable dans le shell avant de lancer la commande). Sans elle, le conteneur démarre avec le mot de passe par défaut de `compose.yaml`, reste « unhealthy » et l'API échoue avec `Login failed for user 'sa'`.

Le mot de passe `sa` n'est pris en compte qu'à la **première** création du volume `database_data`. Si le volume existe déjà avec un autre mot de passe, il faut le supprimer (`docker compose down -v`, **efface les données**) ou utiliser l'ancien mot de passe.

### 4. Lancer l'API

```bash
symfony serve            # ou : php -S 127.0.0.1:8000 -t public
```

L'API répond alors sur `http://127.0.0.1:8000/api`.

## Authentification

1. `POST /api/login` avec `{"username": "...", "password": "..."}` (identifiant et mot de passe AD).
2. La réponse contient `{"token": "<jwt>"}` (valable 1 heure).
3. Envoyer ensuite `Authorization: Bearer <jwt>` sur les routes protégées.

| Code | Signification |
|---|---|
| 401 | Identifiants invalides ou JWT absent / invalide |
| 429 | Trop de tentatives de connexion (5 par IP et identifiant, 30 par IP, sur 15 minutes) |
| 503 | Annuaire LDAP indisponible |

## Routes

`GET` = public. Écritures et `/api/traces` = JWT admin. Les `{id}` sont des entiers.

| Ressource | Routes |
|---|---|
| Services (`libelle`, `localisation`) | `GET /api/services`, `GET /api/services/{id}`, `POST /api/services`, `PUT\|PATCH /api/services/{id}`, `DELETE /api/services/{id}` |
| Métiers (`libelle`, `nom`, `prenom`) | `/api/metiers` (mêmes 5 routes) |
| Numéros d'urgence (`libelle`, `numero`) | `/api/numeros-urgence` (mêmes 5 routes) |
| Personnel de garde (`libelle`, `serviceId`, `metierId`) | `/api/personnel` (mêmes 5 routes) |
| Numéros de garde (`numero`, `type`, `personnelDeGardeId`) | `/api/numeros-garde` (mêmes 5 routes) |
| Annuaire du personnel (`nom`, `prenom`, `email`, `telephone`, `serviceId`, `metierId`) | `/api/personnes` (mêmes 5 routes) |
| Traces (journal d'audit, lecture seule) | `GET /api/traces`, `GET /api/traces/{id}` |
| Connexion | `POST /api/login` |

- `PUT` et `PATCH` sont équivalents : seuls les champs envoyés sont modifiés.
- Les relations se donnent par identifiant (`serviceId`, `metierId`, `personnelDeGardeId`) ; un id inconnu renvoie 400.
- Supprimer un service ou un métier encore utilisé renvoie 409. Supprimer un personnel supprime ses numéros de garde.
- Le détail de chaque route (accès, champs, codes de retour) est dans le commentaire au-dessus de la route,
  dans `src/Controller/Api/`.

### Recherche — `GET /api/personnel` et `GET /api/personnes`

| Paramètre | Description |
|---|---|
| `q` | Mots recherchés (50 caractères max), insensible à la casse, dans le libellé du personnel, le service, sa localisation et le métier. Tous les mots doivent correspondre. |
| `serviceId`, `metierId` | Filtres par identifiant |
| `sort` | Tri : `nom` (défaut) ou `service` |
| `page` | Numéro de page (défaut 1) |
| `limit` | Taille de page (défaut 20, max 100) |

La pagination est renvoyée dans les en-têtes `X-Total-Count`, `X-Page`, `X-Per-Page`, `X-Total-Pages`.
`GET /api/traces` est paginé de la même façon (défaut 50, max 200), du plus récent au plus ancien.

### Format des erreurs

```json
{ "message": "Ressource introuvable." }
```

Erreur de validation (422) :

```json
{ "message": "Données invalides.", "errors": { "libelle": ["This value should not be blank."] } }
```

Autres codes : 400 (JSON ou type invalide, relation inconnue), 401, 404, 409, 500 (message générique hors mode debug).

## Tests de l'API

```bash
php bin/console doctrine:database:create --env=test   # une seule fois : crée ANNUAIRE_IMM_DEV_test
php bin/console doctrine:migrations:migrate --env=test
php bin/phpunit
```

Les tests utilisent une base séparée (suffixe `_test`), vidée avant chaque test, et un JWT généré localement ;
la connexion LDAP est testée avec un faux annuaire, sans réseau.

## Structure de l'API

```
src/Controller/Api/   Contrôleurs CRUD (un par ressource) + AbstractApiController
src/Entity/           Entités Doctrine et groupes de sérialisation
src/Repository/       Requêtes (recherche du personnel, traces paginées)
src/Security/         LdapAuthenticator (login), JwtUserProvider, AdminUser
src/Service/          TraceLogger (journal d'audit transactionnel)
src/EventSubscriber/  ExceptionSubscriber (réponses d'erreur JSON)
migrations/           Schéma de base de données
tests/                Tests fonctionnels (Api/) et unitaires (Security/)
```

---

# Frontend

Interface web (Vue 3, Vue Router, Pinia, Vite, Vitest) qui consomme l'API. Elle se trouve dans le dépôt [Frontend-annuaire](https://github.com/Otto-Cyril/Frontend-annuaire).

## Prérequis

- Node.js
- L'API en cours d'exécution sur `http://127.0.0.1:8000`

## Démarrage et scripts

```bash
npm install
npm run dev
```

| Commande | Effet |
|---|---|
| `npm run dev` | Serveur de développement (http://localhost:5173/) |
| `npm run build` | Build de production dans `dist/` |
| `npm run preview` | Prévisualisation du build |
| `npm test` | Tests Vitest |

## Pages

| Route | Accès | Contenu |
|---|---|---|
| `/` | Public | Personnel de garde, avec leurs numéros de garde (pagination) |
| `/annuaire` | Public | Annuaire du personnel (recherche, filtres, pagination) |
| `/personnel/:id` | Public | Fiche d'une personne |
| `/connexion` | Public | Connexion avec un compte Active Directory |
| `/admin/:resource` | Connexion requise | Gestion des services, métiers, numéros d'urgence, personnel de garde, numéros de garde et annuaire du personnel |
| `/admin/personnel/:id/modifier` | Connexion requise | Modification d'une fiche |
| `/admin/traces` | Connexion requise | Journal des actions |

Les routes d'administration redirigent vers la page de connexion si aucun jeton n'est présent. La barre des numéros d'urgence est affichée sur toutes les pages, et le mode clair/sombre se change depuis l'en-tête.

## Configuration et authentification du frontend

| Variable | Défaut | Rôle |
|---|---|---|
| `VITE_API_URL` | `/api` | Adresse de base de l'API |

La connexion appelle `POST /api/login` et reçoit un jeton JWT, conservé dans le `sessionStorage` (il disparaît à la fermeture de l'onglet) et envoyé dans l'en-tête `Authorization: Bearer`. Un jeton expiré (réponse 401) ramène à la page de connexion.

Le frontend ne distingue pas les rôles : tout compte connecté accède à l'administration. Les droits réels sont contrôlés par l'API.

## Structure du frontend

```
src/
├── api.js            Client HTTP (jeton, pagination, messages d'erreur)
├── router.js         Routes et garde d'accès à l'administration
├── resources.js      Description des ressources administrables
├── theme.js          Mode clair/sombre
├── stores/auth.js    Session (Pinia)
├── composables/      Logique de l'annuaire (recherche, filtres)
├── components/       Barre d'urgence, filtres, pagination, icônes
├── views/            Une vue par page
└── style.css         Styles globaux
```

## Déploiement du frontend

`npm run build` produit le dossier `dist/`, à servir comme site statique. Le serveur web doit :

- relayer `/api` vers l'API (ou définir `VITE_API_URL` avant le build) ;
- renvoyer `index.html` pour les routes inconnues, car le routeur utilise l'historique du navigateur.
