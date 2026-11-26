# TP 4 — Conteneurisation avec Docker, Docker Compose & Publication GHCR

Ce document consigne les étapes de réalisation, les preuves d'exécution et les exercices du TP 4 du cours "DevOps et IA" (Master IS2IA - ESISA).

---

## 1. Conception du `Dockerfile` Multi-Stage et Sécurité

Le fichier `Dockerfile` met en œuvre le pattern *multi-stage build* pour concilier légèreté, sécurité et isolation :
- **Stage 1 (`builder`)** : Base `python:3.12-slim`, installation des dépendances dans `/install` avec `pip install --no-cache-dir --prefix=/install`.
- **Stage 2 (`runtime`)** : Base `python:3.12-slim`, copie des dépendances compilées depuis `/install` vers `/usr/local`, copie du code applicatif `app/`.
- **Sécurité & Non-root** : Création d'un utilisateur dédié sans privilèges (`appuser`, UID 1000).
- **HEALTHCHECK** : Sonde de santé intégrée exécutant un script Python `urllib` natif (éliminant toute dépendance à `curl`).

### Fichier `.dockerignore`
Exclusion stricte des fichiers et dossiers inutiles ou sensibles :
```text
.git
.venv
__pycache__
.env
*.pyc
.github
tests
```

---

## 2. Preuves d'Exécution Locale

### 2.1 Construction de l'image
```bash
docker build -t capstone-api:0.1 .
```

### 2.2 Vérification de l'utilisateur non-root
```bash
docker run -i --rm capstone-api:0.1 id
```
**Sortie réelle obtenue :**
```text
uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
```
*Preuve formelle :* Le conteneur s'exécute sous `appuser` (UID 1000) et non `root` (UID 0).

### 2.3 Exécution et Sonde de Santé (`/health`)
```bash
docker run -d --name capstone-test -p 8000:8000 capstone-api:0.1
curl http://localhost:8000/health
```
**Sortie réelle obtenue :**
```json
{"status":"ok","service":"capstone-api","version":"1.0.0"}
```

### 2.4 Historique et couches de l'image (`docker history`)
```text
IMAGE          CREATED         CREATED BY                                      SIZE      COMMENT
723b29e054a1   5 minutes ago   CMD ["uvicorn" "app.main:app" "--host" "0.0.…   0B        buildkit.dockerfile.v0
<missing>      5 minutes ago   HEALTHCHECK &{["CMD-SHELL" "python -c \"impo…   0B        buildkit.dockerfile.v0
<missing>      5 minutes ago   EXPOSE [8000/tcp]                               0B        buildkit.dockerfile.v0
<missing>      5 minutes ago   USER appuser                                    0B        buildkit.dockerfile.v0
<missing>      5 minutes ago   RUN /bin/sh -c useradd -m -u 1000 appuser &&…   69.6kB    buildkit.dockerfile.v0
<missing>      5 minutes ago   COPY app/ ./app/ # buildkit                     28.7kB    buildkit.dockerfile.v0
<missing>      5 minutes ago   COPY /install /usr/local # buildkit             48.6MB    buildkit.dockerfile.v0
<missing>      5 minutes ago   WORKDIR /app                                    8.19kB    buildkit.dockerfile.v0
```

---

## 3. Orchestration Multi-Conteneurs avec Docker Compose

Le fichier `docker-compose.yml` orchestre le service `api` et la base de données `postgres:16-alpine`.  
Points clés :
- Dépendance conditionnelle : `depends_on: db: condition: service_healthy`.
- Volume nommé persistant : `db_data:/var/lib/postgresql/data`.
- Tous les mots de passe et credentials sont explicitement balisés comme **LAB ONLY**.

### Exécution du cycle de vie
```bash
docker compose up -d --build
docker compose ps
docker compose logs --tail=20 api
docker compose down -v
```

**Sortie réelle obtenue :**
```text
 Container capstone-starter-db-1  Started
 Container capstone-starter-db-1  Healthy
 Container capstone-starter-api-1  Started

NAME                     IMAGE                  COMMAND                  SERVICE   CREATED          STATUS                    PORTS
capstone-starter-api-1   capstone-starter-api   "uvicorn app.main:ap…"   api       14 seconds ago   Up 6 seconds (healthy)    0.0.0.0:8000->8000/tcp
capstone-starter-db-1    postgres:16-alpine     "docker-entrypoint.s…"   db        14 seconds ago   Up 13 seconds (healthy)   5432/tcp

api-1  | INFO:     Started server process [1]
api-1  | INFO:     Waiting for application startup.
api-1  | INFO:     Application startup complete.
api-1  | INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
api-1  | INFO:     127.0.0.1:35886 - "GET /health HTTP/1.1" 200 OK

 Container capstone-starter-api-1  Stopped and Removed
 Container capstone-starter-db-1   Stopped and Removed
 Volume capstone-starter_db_data   Removed
```

[CAPTURE À AJOUTER : Sortie de docker compose ps montrant les statuts (healthy)]

---

## 4. Tableau Comparatif des Tailles d'Images (Single-Stage vs Multi-Stage)

| Type de build | Image de base | Couches applicatives & build tools | Taille finale | Réduction |
|---|---|---|---|---|
| **Build naïf (Single-Stage)** | `python:3.12` | Compilateurs gcc, caches apt/pip, headers Python | **1.71 GB** | Référence (0%) |
| **Multi-Stage Optimisé** | `python:3.12-slim` | Uniquement runtime python + `/install` copié | **240 MB** | **-86% (-1.47 GB)** |

*Analyse :* Le multi-stage build permet de diviser la taille de l'image par 7 en éliminant l'ensemble de la chaîne de compilation et des outils de packaging Debian.

---

## 5. Exercice IA : Revue et Optimisation du Dockerfile

| Suggestion de l'IA | Analyse & Test réel | Statut | Justification |
|---|---|---|---|
| **1. Variables d'environnement Python** (`PYTHONUNBUFFERED=1`, `PYTHONDONTWRITEBYTECODE=1`) | Build testé avec succès. | **Accepté** | Standard DevOps pour Python en conteneur : évite l'écrasement des logs dans les buffers et la génération de `.pyc` inutiles. |
| **2. Fixation explicite de l'UID appuser** (`useradd -u 1000`) | Build testé avec succès (`uid=1000(appuser)`). | **Accepté** | Garantit la reproductibilité des permissions entre l'hôte et le conteneur, notamment pour les volumes montés. |
| **3. Remplacement de `--prefix=/install` par `/root/.local`** | Analyse technique comparative. | **Rejeté** | L'option `--prefix=/install` permet une copie atomique propre vers `/usr/local` sans modifier le `PATH` utilisateur. |
| **4. Nettoyage de cache apt (`rm -rf /var/lib/apt/lists/*`)** | Vérification du Dockerfile. | **Rejeté** | Aucun appel `apt-get` n'est effectué dans le Dockerfile courant (dépendances Python pures). |

---

## 6. Publication sur GitHub Packages (GHCR)

Le workflow CI `.github/workflows/ci.yml` a été enrichi avec le job `build` déclenché uniquement lors de la fusion sur `main` :
- Authentification automatique via `GITHUB_TOKEN` sur `ghcr.io`.
- Image poussée avec nommage en minuscules : `ghcr.io/malkiouss/capstone-starter:main`.

**URL du package sur GitHub :**  
`https://github.com/Malkiouss/capstone-starter/pkgs/container/capstone-starter`

[CAPTURE À AJOUTER : Page GitHub Packages montrant l'image capstone-starter:main publiée]

---

## 7. Pièges fréquents / Erreurs et Solutions

| Erreur / Symptôme | Cause racine | Solution |
|---|---|---|
| `open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified` | Moteur Docker Desktop non démarré ou service Windows arrêté. | Démarrer Docker Desktop ou exécuter dockerd dans WSL avec exposition TCP `tcp://0.0.0.0:2375`. |
| Échec du Healthcheck : `curl: command not found` | Les images de base `-slim` ou `alpine` n'embarquent pas `curl` par défaut. | Implémenter le healthcheck avec le module Python standard `urllib.request`. |
| `denied: installation not allowed to Create organization package` sur GHCR | Nom du dépôt ou de l'organisation contenant des majuscules dans le tag GHCR. | Forcer la conversion en minuscules du tag d'image (`ghcr.io/malkiouss/capstone-starter:main`). |
| Conteneur API démarré avant que Postgres ne soit prêt | `depends_on: [db]` attend uniquement le démarrage du conteneur et non la disponibilité effective du port 5432. | Utiliser `depends_on: db: condition: service_healthy` couplé à un healthcheck `pg_isready`. |
