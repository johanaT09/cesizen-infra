# CESIZen — Environnement Docker local

Ce dépôt (`cesizen-infra`) contient l'orchestration Docker Compose du projet CESIZen.
Il ne contient pas de code applicatif : le backend est dans `cesizen-api`, le frontend
dans `cesizen-web`. Ce README documente toutes les commandes utiles pour faire tourner,
réinitialiser et déboguer l'environnement local.

## Prérequis

Les trois dépôts doivent être clonés **au même niveau** sur le disque :

```
un-dossier-parent/
├── cesizen-infra/     (ce dépôt)
├── cesizen-api/       (backend Laravel)
└── cesizen-web/       (frontend Nuxt)
```

Docker Desktop doit être lancé avant toute commande `docker compose`.

## Premier démarrage (une seule fois)

1. Créer le fichier d'environnement du backend :
   ```bash
   cd ../cesizen-api
   cp .env.example .env
   ```
   Vérifier que les variables de connexion à la base pointent bien vers le service
   Docker (pas `127.0.0.1`) :
   ```
   DB_CONNECTION=pgsql
   DB_HOST=db
   DB_PORT=5432
   DB_DATABASE=cesizen
   DB_USERNAME=cesizen
   DB_PASSWORD=secret
   ```

2. Démarrer les conteneurs :
   ```bash
   cd ../cesizen-infra
   docker compose up --build
   ```

3. Dans un second terminal, générer la clé d'application Laravel (obligatoire,
   sinon erreur 500 immédiate) :
   ```bash
   docker compose exec app php artisan key:generate
   ```

4. Créer les tables :
   ```bash
   docker compose exec app php artisan migrate
   ```

5. Peupler la base avec des données de démo :
   ```bash
   docker compose exec app php artisan db:seed
   ```

6. Vérifier que tout répond :
   - Frontend : http://localhost:3000
   - API (via Nginx) : http://localhost:8080
   - Mailpit (emails interceptés) : http://localhost:8025

## Commandes du quotidien

| Action | Commande |
|---|---|
| Démarrer tous les services | `docker compose up` |
| Démarrer en reconstruisant les images | `docker compose up --build` |
| Démarrer en arrière-plan | `docker compose up -d` |
| Arrêter les services | `docker compose down` |
| Arrêter et supprimer aussi les volumes (⚠️ efface la base) | `docker compose down -v` |
| Voir les logs de tous les services | `docker compose logs -f` |
| Voir les logs d'un seul service | `docker compose logs -f app` (ou `front`, `db`, `nginx`) |
| Reconstruire un seul service | `docker compose up -d --build front` |
| Ouvrir un shell dans le conteneur backend | `docker compose exec app sh` |
| Ouvrir un shell dans le conteneur frontend | `docker compose exec front sh` |

## Base de données

| Action | Commande |
|---|---|
| Lancer les migrations | `docker compose exec app php artisan migrate` |
| Lancer le seeding (données de démo) | `docker compose exec app php artisan db:seed` |
| Lancer un seeder précis | `docker compose exec app php artisan db:seed --class=NomDuSeeder` |
| Tout réinitialiser (tables + seed) | `docker compose exec app php artisan migrate:fresh --seed` |
| Voir la liste des seeders disponibles | `docker compose exec app ls database/seeders/` |
| Se connecter directement à Postgres (CLI) | `docker compose exec db psql -U cesizen -d cesizen` |

**À utiliser avant la soutenance** pour repartir d'un état propre et démontrable :
```bash
docker compose exec app php artisan migrate:fresh --seed
```

## Backend Laravel — commandes utiles

| Action | Commande |
|---|---|
| Voir toutes les routes API | `docker compose exec app php artisan route:list --path=api` |
| Générer une clé d'application | `docker compose exec app php artisan key:generate` |
| Vider les caches (config, routes, vues) | `docker compose exec app php artisan optimize:clear` |
| Lancer les tests | `docker compose exec app php artisan test` |

## Dépannage — problèmes déjà rencontrés

**`npm ci` échoue avec "Missing: X from lock file" (côté cesizen-web)**
Le `package-lock.json` n'est plus synchronisé avec `package.json`. Sur la machine
(hors conteneur) :
```bash
cd ../cesizen-web
rm -rf node_modules package-lock.json
npm install
```
Vérifier ensuite qu'aucun autre fichier (ex. `nuxt.config.ts`) n'a été modifié
involontairement avant de committer.

**Erreur `vendor/autoload.php: No such file or directory` (côté cesizen-api)**
Le volume Docker monte le dossier local par-dessus l'image, ce qui masque le
`vendor/` installé pendant le build. Corrigé dans `docker-compose.yml` par un
volume nommé dédié (`app-vendor:/var/www/html/vendor`) — si l'erreur revient,
vérifier que ce volume est bien présent dans le fichier compose.

**Erreur 500 générique sans détail**
Vérifier les logs pour le vrai message :
```bash
docker compose logs app --tail=50
```
Causes les plus fréquentes : `.env` absent, `APP_KEY` non généré, migrations
non exécutées (table `sessions` ou autre manquante).

**Le frontend appelle l'API mais reçoit des 404 sur toutes les routes**
Vérifier que `NUXT_PUBLIC_API_BASE` inclut bien le préfixe `/api` attendu par
Laravel (`http://localhost:8080/api`, pas juste `http://localhost:8080`).

**Message Nginx `can not modify /etc/nginx/conf.d/default.conf (read-only file system?)`**
Sans conséquence — Nginx tente de patcher sa config automatiquement mais on lui
fournit déjà la nôtre en lecture seule. Aucune action nécessaire.

## URLs de référence

| Service | URL |
|---|---|
| Frontend (Nuxt) | http://localhost:3000 |
| API backend (via Nginx) | http://localhost:8080 |
| Mailpit (emails de test) | http://localhost:8025 |
| PostgreSQL (accès direct, ex. DBeaver) | localhost:5432 |
