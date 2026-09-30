# Frontend — Annuaire des gardes et astreintes

Interface web de l'annuaire du personnel de garde de l'Institut Mutualiste Montsouris (IMM) : recherche du personnel, personnel de garde, numéros d'urgence et administration. Elle consomme l'API REST du dépôt [API-REST-annuaire](https://github.com/Otto-Cyril/API-REST-annuaire).

Technologies : Vue 3, Vue Router, Pinia, Vite, Vitest.

## Prérequis

- Node.js
- L'API en cours d'exécution (voir le README du dépôt de l'API)

## Démarrage

```bash
npm install
npm run dev
```

L'application est servie sur http://localhost:5173/. En développement, Vite relaie `/api` vers `http://127.0.0.1:8000` (`vite.config.js`) : l'API doit donc tourner sur ce port, par exemple :

```bash
php -S 127.0.0.1:8000 -t public
```

## Scripts

| Commande | Effet |
|---|---|
| `npm run dev` | Serveur de développement |
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

## Configuration

| Variable | Défaut | Rôle |
|---|---|---|
| `VITE_API_URL` | `/api` | Adresse de base de l'API |

## Authentification

La connexion appelle `POST /api/login` et reçoit un jeton JWT, conservé dans le `sessionStorage` (il disparaît à la fermeture de l'onglet) et envoyé dans l'en-tête `Authorization: Bearer`. Un jeton expiré (réponse 401) ramène à la page de connexion.

Le frontend ne distingue pas les rôles : tout compte connecté accède à l'administration. Les droits réels sont contrôlés par l'API.

## Structure

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

## Déploiement

`npm run build` produit le dossier `dist/`, à servir comme site statique. Le serveur web doit :

- relayer `/api` vers l'API (ou définir `VITE_API_URL` avant le build) ;
- renvoyer `index.html` pour les routes inconnues, car le routeur utilise l'historique du navigateur.
