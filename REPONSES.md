# TP2 — Orchestration d'une application multi-conteneurs avec Docker Compose

## Partie 1 — Variables d'environnement et configuration

### Q1 — Variables d'environnement nécessaires à Strapi

Strapi a besoin des variables suivantes pour se connecter à PostgreSQL :

- `DATABASE_CLIENT` : système de gestion de base de données utilisé, ici `postgres`.
- `DATABASE_HOST` : nom du service PostgreSQL dans Docker Compose, ici `postgres`.
- `DATABASE_PORT` : port interne de PostgreSQL, généralement `5432`.
- `DATABASE_NAME` : nom de la base de données.
- `DATABASE_USERNAME` : nom de l'utilisateur PostgreSQL.
- `DATABASE_PASSWORD` : mot de passe de cet utilisateur.

On utilise `DATABASE_HOST=postgres` parce que Docker Compose fournit une résolution DNS entre les services. `localhost` désignerait le conteneur Strapi lui-même, et non le conteneur PostgreSQL.

### Q2 — Différence entre `.env` et `.env.example`

Le fichier `.env.example` est un modèle qui présente les variables d'environnement nécessaires au fonctionnement de l'application. Il est versionné dans Git, à condition de ne pas contenir de véritables secrets.

Le fichier `.env` contient les valeurs réellement utilisées localement, notamment les mots de passe. Il doit être ajouté à `.gitignore` afin d'éviter de publier des informations sensibles.

## Partie 2 — Docker Compose et persistance

### Q3 — Différence entre `docker compose down` et `docker compose down -v`

- `docker compose down` arrête et supprime les conteneurs ainsi que le réseau créé par Compose, mais **conserve** les volumes nommés.
- `docker compose down -v` supprime **également** les volumes nommés associés au projet.

La seconde commande peut donc supprimer les données persistantes de PostgreSQL, les fichiers téléversés dans Strapi et les données de pgAdmin. Elle doit être utilisée avec prudence.

### Q4 — Pourquoi utiliser `$${POSTGRES_USER}` dans un healthcheck ?

Dans un fichier Docker Compose, `${POSTGRES_USER}` est normalement interprété par Compose lors de la lecture du fichier.

Le double dollar `$$` empêche cette interpolation immédiate. Le conteneur reçoit alors `${POSTGRES_USER}`, que le shell développe au moment de l'exécution.

Cela permet au healthcheck d'utiliser la variable d'environnement disponible dans le conteneur PostgreSQL.

## Partie 3 — Configuration de pgAdmin

### Q5 — Configuration de `servers.json`

**1. Pourquoi utiliser `"Host": "postgres"` ?**

`postgres` correspond au nom du service PostgreSQL dans Docker Compose. Les services connectés au même réseau Docker peuvent communiquer en utilisant leur nom de service.

**2. Que signifie `:ro` dans un volume ?**

`ro` signifie _read-only_ (lecture seule). Le conteneur peut lire les fichiers montés, mais ne peut pas les modifier par ce montage.

**3. Que signifie `${PGADMIN_PORT:-5050}:80` ?**

Le port `80` est le port du conteneur pgAdmin. Il est associé à un port de la machine hôte :

- si `PGADMIN_PORT` est défini, sa valeur est utilisée ;
- sinon, le port `5050` est utilisé par défaut.

pgAdmin est donc accessible à l'adresse `http://localhost:5050` lorsque le port par défaut est utilisé.

## Partie 4 — Dockerfile de Strapi

### Q6 — Pourquoi copier les fichiers de dépendances avant le reste du code ?

Le Dockerfile utilise l'ordre suivant :

```dockerfile
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
```

On copie d'abord `package.json` et `package-lock.json`, puis on exécute `npm ci` pour installer les dépendances. Ensuite, on copie le reste du code et on lance la compilation.

Docker met en cache les couches de construction. Si seul le code source change, la couche d'installation des dépendances est réutilisée, ce qui accélère la reconstruction de l'image.

### Q7 — Pourquoi utiliser `USER node` et `--chown=node:node` ?

`USER node` permet d'exécuter l'application avec l'utilisateur non privilégié `node` au lieu de `root`. Cela réduit les risques en cas de compromission de l'application.

L'option `--chown=node:node` définit le propriétaire et le groupe des fichiers copiés. L'utilisateur `node` peut ainsi lire (et écrire si nécessaire) les fichiers dont il a besoin, par exemple pour les fichiers téléversés ou le cache de build.

## Partie 5 — Frontend Next.js

### Q8 — Pourquoi attendre que PostgreSQL soit healthy avant de démarrer Strapi ?

Un simple ordre de démarrage ne garantit pas que PostgreSQL est prêt à accepter les connexions : le conteneur peut être démarré alors que la base est encore en cours d'initialisation.

Avec `depends_on` et `condition: service_healthy`, Docker Compose attend que le healthcheck de PostgreSQL réussisse avant de démarrer Strapi. Cela évite les erreurs de connexion au démarrage.

### Q9 — Comment vérifier les articles dans PostgreSQL ?

Ouvrir un terminal PostgreSQL dans le conteneur :

```cmd
docker compose exec postgres psql -U strapi -d strapi
```

Afficher les tables :

```sql
\dt
```

Rechercher les tables liées aux articles :

```sql
SELECT tablename
FROM pg_tables
WHERE schemaname = 'public'
  AND tablename ILIKE '%article%';
```

**Résultat observé :** la requête retourne une seule ligne, la table `articles` (voir la capture pgAdmin).

Pour compter les articles publiés (Strapi v5 : les entrées publiées ont `published_at` non nul) :

```sql
SELECT COUNT(*) FROM articles WHERE published_at IS NOT NULL;
```

**Résultat observé :** le frontend affiche **3 articles publiés** :

1. L'intelligence artificielle transforme notre quotidien
2. Les bonnes pratiques pour protéger ses données en ligne
3. Le Data Engineering

Ces articles sont bien lus depuis Strapi, donc stockés dans PostgreSQL.

### Q10 — Pourquoi utiliser `http://strapi:1337` plutôt que `http://localhost:1337` dans le frontend ?

Depuis le conteneur frontend, `localhost` désigne le frontend lui-même, pas le conteneur Strapi.

L'adresse `http://strapi:1337` permet au frontend de joindre Strapi grâce au DNS interne de Docker Compose.

### Q11 — Combien de stages contient le Dockerfile du frontend ? Pourquoi `output: "standalone"` ?

Le Dockerfile contient **trois stages** :

1. `deps` : installe les dépendances.
2. `builder` : construit l'application Next.js.
3. `runner` : exécute l'application en production.

L'option `output: "standalone"` fait générer à Next.js un ensemble autonome contenant le serveur et uniquement les dépendances nécessaires à l'exécution. L'image de production est ainsi beaucoup plus légère.

Le stage final copie `.next/standalone`, `.next/static` et `public`. Sans la sortie standalone, le fichier `server.js` utilisé par la commande de démarrage n'existerait pas à cet emplacement.

Pour examiner la taille de l'image construite :

```cmd
docker images tp2-frontend
```

## Partie 6 — Réseaux Docker et sécurité

### Q12 — Pourquoi Strapi doit-il appartenir aux deux réseaux ?

Strapi doit communiquer avec PostgreSQL (accès aux données) et avec le frontend (fourniture de l'API).

Il est donc connecté au réseau **backend** (communication avec PostgreSQL) et au réseau **frontend** (communication avec Next.js). Il sert de passerelle : le frontend n'a jamais d'accès direct à la base.

PostgreSQL n'a pas besoin d'un port publié sur la machine hôte : les autres services le joignent directement sur le réseau Docker interne, ce qui réduit son exposition.

### Q13 — Donner deux mesures de sécurité

**Mesure 1 : exécuter les applications avec un utilisateur non privilégié.**
`USER node` limite les privilèges du processus et réduit les conséquences d'une compromission.

**Mesure 2 : protéger les identifiants et les mots de passe.**
Les secrets sont conservés dans `.env`, exclu de Git, et ne sont jamais publiés dans le dépôt. On utilise des mots de passe forts et uniques.

Mesure complémentaire : ne pas publier le port PostgreSQL sur la machine hôte quand ce n'est pas nécessaire.

## Partie 7 — Cycle de vie et maintenance

### Q14 — Que signifie `restart: unless-stopped` ?

Docker redémarre automatiquement Strapi lorsqu'il s'arrête de manière inattendue (crash) ou après le redémarrage du démon Docker, sauf si le conteneur a été arrêté manuellement.

- Après `docker compose stop strapi`, Strapi **ne redémarre pas** automatiquement : Docker retient l'arrêt manuel.
- Après `docker compose kill strapi`, le processus est tué (code de sortie 137) ; Docker le traite comme un arrêt inattendu et applique la politique de redémarrage, donc Strapi est relancé.

Pour relancer manuellement Strapi après un `stop` :

```cmd
docker compose start strapi
```

## Captures d'écran

- **Frontend** (`http://localhost:3000`) : affiche les 3 articles ci-dessus (`les_articles_avec_frontend.png`).
- **pgAdmin** (`http://localhost:5050`) : la requête sur `pg_tables` retourne la table `articles` dans la base `strapi` (`image.png`).
