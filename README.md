# Application Flask + MariaDB — Pipeline CI/CD

Petite application web **Flask** qui se connecte à une base **MariaDB**.
La page d'accueil indique si la connexion à la base fonctionne :

- `Database Connection Status: Connected` si la base répond ;
- `Database Connection Status: Disconnected` sinon.

## Prérequis

- [Docker](https://docs.docker.com/get-docker/)
- (Optionnel) Python 3.13 pour un lancement sans Docker

## Variables d'environnement

| Variable      | Rôle            | Valeur par défaut |
|---------------|-----------------|-------------------|
| `DB_HOST`     | Hôte de la base | `localhost`       |
| `DB_USER`     | Utilisateur     | `root`            |
| `DB_PASSWORD` | Mot de passe    | *(vide)*          |
| `DB_NAME`     | Nom de la base  | `test`            |

## Lancer l'application avec Docker

1. Démarrer une base MariaDB :

   ```bash
   docker network create appnet

   docker run -d --name db --network appnet \
     -e MYSQL_ROOT_PASSWORD=rootpass \
     -e MYSQL_DATABASE=test \
     mariadb:11
   ```

2. Construire et lancer l'application :

   ```bash
   docker build -t flask-app .

   docker run -d --name app --network appnet -p 5000:5000 \
     -e DB_HOST=db \
     -e DB_USER=root \
     -e DB_PASSWORD=rootpass \
     -e DB_NAME=test \
     flask-app
   ```

3. Ouvrir http://localhost:5000 → la page affiche l'état de la connexion.

## Lancer l'application depuis l'image GHCR

```bash
docker pull ghcr.io/kylian2210/evalfinal:latest
docker run -p 5000:5000 ghcr.io/kylian2210/evalfinal:latest
```

## Lancer sans Docker (développement)

```bash
python -m venv .venv
source .venv/bin/activate      # Windows : .venv\Scripts\activate
pip install -r requirements.txt
python app.py
```

---

## Intégration et déploiement continus (CI/CD)

### Publication de l'image sur GHCR — `.github/workflows/publish.yml`

- Se déclenche à **chaque push sur `main`** et sur les **tags `v*.*.*`**.
- Construit l'image Docker et la publie sur `ghcr.io/kylian2210/evalfinal`.
- Étiquettes produites : `latest` (sur `main`), `sha-<commit>` (SHA du commit) et `1.0.0` (depuis un tag git `v1.0.0`).
- **Aucun mot de passe** dans le dépôt : authentification via le `GITHUB_TOKEN` fourni automatiquement, avec la permission `packages: write`.

Poser la version `v1.0.0` :

```bash
git tag v1.0.0
git push origin v1.0.0
```

Puis, dans l'onglet **Packages** du dépôt → *Package settings* → **Change visibility** → rendre le package **public**, pour qu'un `docker pull` fonctionne sans compte.

### Test fonctionnel avant merge — `.github/workflows/test.yml`

- Se déclenche à **chaque Pull Request vers `main`**.
- Démarre une base **MariaDB**, construit puis lance l'application, appelle la page d'accueil et vérifie qu'elle répond `Database Connection Status: Connected`.
- Le job **échoue** si la connexion échoue.

### Protection de la branche `main`

Dans **Settings → Branches** sur `main` : *Require a pull request before merging* + *Require status checks to pass before merging* en sélectionnant le check **`functional-test`**. Le bouton **Merge** reste alors bloqué tant que le test n'est pas au vert.

## Convention de commits

Les messages suivent le format [Conventional Commits](https://www.conventionalcommits.org/) : `feat:`, `fix:`, `ci:`, `docs:`, `build:`, `chore:`, etc.
