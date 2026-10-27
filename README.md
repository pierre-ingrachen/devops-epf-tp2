# TP2 – DevOps : Docker, GitHub Actions & Ansible

Application web 3 tiers (front Vue.js, API Spring Boot, base PostgreSQL) conteneurisée avec Docker, testée et publiée par une pipeline GitHub Actions, puis déployée automatiquement sur un serveur distant avec Ansible.

## Sommaire

- [Architecture](#architecture)
- [Structure du dépôt](#structure-du-dépôt)
- [Lancer en local](#lancer-en-local)
- [Intégration continue (CI/CD)](#intégration-continue-cicd)
- [Déploiement avec Ansible](#déploiement-avec-ansible)
- [Secrets GitHub requis](#secrets-github-requis)

## Architecture

```
                 ┌───────────────────── réseau Docker : my-network ─────────────────────┐
                 │                                                                       │
Client ──:80──▶  │  httpd (Apache, reverse proxy)                                        │
                 │     ├── /api/* ──▶ simple-api (Spring Boot, :8080) ──▶ database       │
                 │     └── /*     ──▶ front (Nginx + Vue.js, :80)         (PostgreSQL)   │
                 └───────────────────────────────────────────────────────────────────────┘
```

| Service      | Dossier        | Image de base               | Rôle                                         |
|--------------|----------------|-----------------------------|----------------------------------------------|
| `httpd`      | `http-server/` | Apache httpd                | Reverse proxy, seul service exposé (port 80) |
| `front`      | `front/`       | Node 12 (build) + Nginx     | Interface Vue.js                             |
| `simple-api` | `simple-api/`  | Maven + Amazon Corretto 21  | API REST Spring Boot                         |
| `database`   | `database/`    | PostgreSQL 14.1 (Alpine)    | Base de données, initialisée via `sql/`      |

Seul `httpd` publie un port ; les autres services ne sont joignables que depuis le réseau Docker interne. Les données PostgreSQL sont conservées dans le volume `db-volume`.

## Structure du dépôt

```
tp2/
├── .github/workflows/main.yml   # Pipeline CI/CD
├── ansible/                     # Déploiement (playbook, inventaire, rôles)
├── database/                    # Dockerfile + scripts SQL d'initialisation
├── front/                       # Application Vue.js + Dockerfile + config Nginx
├── http-server/                 # Dockerfile + configuration Apache (reverse proxy)
├── simple-api/                  # API Spring Boot + Dockerfile
└── docker-compose.yaml          # Orchestration locale
```

## Lancer en local

Prérequis : Docker et Docker Compose.

```bash
docker compose up --build
```

Puis :

- Front : <http://localhost/>
- API : <http://localhost/api/departments/IRC/students>

Pour tout arrêter (et supprimer les données) :

```bash
docker compose down -v
```

## Intégration continue (CI/CD)

La pipeline [`.github/workflows/main.yml`](.github/workflows/main.yml) s'exécute à chaque push sur `main` / `develop` et sur chaque pull request.

| Job                          | Déclencheur           | Actions                                                                                         |
|------------------------------|-----------------------|-------------------------------------------------------------------------------------------------|
| `test-backend`               | toujours              | Build et tests Maven (JDK 21), analyse qualité SonarCloud                                       |
| `build-and-push-docker-image`| après `test-backend`  | Build des 4 images ; push sur DockerHub **uniquement sur `main`**                               |
| `deploy`                     | `main` uniquement     | Déploiement Ansible sur le serveur, puis vérification par `curl` de l'API et du front           |

Images publiées sur DockerHub (tag `latest`) :

- `<utilisateur>/tp-devops-simple-api`
- `<utilisateur>/tp-devops-database`
- `<utilisateur>/tp-devops-httpd`
- `<utilisateur>/tp-devops-front`

## Déploiement avec Ansible

Le dossier [`ansible/`](ansible) configure un serveur Debian et y lance les conteneurs.

Rôles, exécutés dans l'ordre par [`playbook.yml`](ansible/playbook.yml) :

| Rôle              | Tag        | Description                                                         |
|-------------------|------------|---------------------------------------------------------------------|
| `install_docker`  | `docker`   | Installe Docker CE et le SDK Python Docker (dans un venv)          |
| `create_network`  | `network`  | Crée le réseau Docker `my-network`                                  |
| `launch_database` | `database` | Lance PostgreSQL avec son volume persistant                         |
| `launch_app`      | `app`      | Lance l'API Spring Boot                                             |
| `launch_front`    | `front`    | Lance le front                                                      |
| `launch_proxy`    | `proxy`    | Lance Apache en reverse proxy (port 80)                             |

Déploiement manuel :

```bash
cd ansible
ansible-playbook -i inventories/setup.yml playbook.yml
```

Pour ne rejouer qu'une partie (par exemple l'API) :

```bash
ansible-playbook -i inventories/setup.yml playbook.yml --tags app
```

Les variables (utilisateur SSH, compte DockerHub, identifiants de la base…) se règlent dans [`ansible/inventories/setup.yml`](ansible/inventories/setup.yml).

## Secrets GitHub requis

À définir dans *Settings → Secrets and variables → Actions* :

| Secret               | Utilisation                                  |
|----------------------|----------------------------------------------|
| `SONAR_TOKEN`        | Authentification auprès de SonarCloud        |
| `DOCKERHUB_USERNAME` | Compte DockerHub                             |
| `DOCKERHUB_TOKEN`    | Jeton d'accès DockerHub                      |
| `SSH_PRIVATE_KEY`    | Clé privée pour se connecter au serveur      |
| `SSH_HOST`           | Adresse du serveur de déploiement            |
