# TP3 - Ansible

## Etat final

Le serveur `ayman.benhadid.takima.school` est gere par Ansible avec un inventaire dedie dans `ansible/inventories/setup.yml`.

Le playbook principal est `ansible/playbook.yml` et applique les roles suivants :

- `docker`
- `build_images`
- `network`
- `database`
- `app`
- `proxy`

Les conteneurs deployes sur le serveur sont :

```text
http-server   aymanatics/tp-devops-http-server:latest   0.0.0.0:80->80/tcp
backend-api   aymanatics/tp-devops-simple-api:latest    8080/tcp
postgres-db   aymanatics/tp-devops-postgres-db:latest   5432/tcp
```

Verification publique :

```bash
curl -fsSI http://ayman.benhadid.takima.school/
curl -fsS http://ayman.benhadid.takima.school/students
```

Resultat API observe :

```json
[{"id":1,"firstname":"Eli","lastname":"Copter","department":{"id":1,"name":"IRC"}},{"id":2,"firstname":"Emma","lastname":"Carena","department":{"id":2,"name":"ETI"}},{"id":3,"firstname":"Jack","lastname":"Uzzi","department":{"id":2,"name":"ETI"}},{"id":4,"firstname":"Aude","lastname":"Javel","department":{"id":3,"name":"CGP"}}]
```

## 3-1 Inventory et commandes de base

Inventaire :

```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: ../id_rsa
    ansible_python_interpreter: /usr/bin/python3
  children:
    prod:
      hosts:
        ayman.benhadid.takima.school:
          ansible_host: ayman.benhadid.takima.school
```

Commandes utilisees :

```bash
ANSIBLE_HOME=.ansible ansible all -m ping
ANSIBLE_HOME=.ansible ansible all -m setup -a "filter=ansible_distribution*"
ANSIBLE_HOME=.ansible ansible-playbook playbook.yml --syntax-check
```

Facts observes :

```text
Debian 12.7 bookworm
```

## 3-2 Playbook

Le playbook installe Docker depuis le depot officiel Docker pour Debian, cree un virtualenv `/opt/docker_venv`, installe le SDK Python Docker, puis lance les roles applicatifs.

La syntaxe a ete verifiee avec :

```bash
ANSIBLE_HOME=.ansible ansible-playbook playbook.yml --syntax-check
```

## 3-3 Configuration docker_container

Les conteneurs sont lances avec `community.docker.docker_container`.

Base de donnees :

- image : `{{ dockerhub_username }}/tp-devops-postgres-db:{{ image_tag }}`
- nom : `postgres-db`
- volume : `tp3-postgres-data:/var/lib/postgresql/data`
- variables : `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`
- reseau : `tp3-app-network`

API :

- image : `{{ dockerhub_username }}/tp-devops-simple-api:{{ image_tag }}`
- nom : `backend-api`
- variables : connexion PostgreSQL
- reseau : `tp3-app-network`

Proxy et front :

- image : `{{ dockerhub_username }}/tp-devops-http-server:{{ image_tag }}`
- nom : `http-server`
- port public : `80:80`
- reseau : `tp3-app-network`

Le proxy Apache sert le front statique et redirige les routes API vers `backend-api:8080`.

## Continuous Deployment

Le workflow `.github/workflows/docker-publish.yml` :

- attend le succes de `Backend CI`
- ne deploie que depuis `main`
- publie les trois images Docker sur DockerHub
- tague chaque image avec `latest` et le SHA Git
- lance Ansible sur le serveur avec `IMAGE_TAG=<sha>`

Secrets GitHub requis :

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
- `ANSIBLE_SSH_PRIVATE_KEY`
- `POSTGRES_PASSWORD`
- `SONAR_TOKEN` si SonarCloud est utilise

Variables GitHub utiles :

- `POSTGRES_DB`
- `POSTGRES_USER`
- `SONAR_PROJECT_KEY`
- `SONAR_ORGANIZATION`

## Securite du deploiement automatique

Deployer automatiquement chaque nouvelle image `latest` du hub n'est pas totalement sur.

Risques :

- une image peut etre remplacee sous le meme tag
- un push DockerHub compromis peut declencher un deploiement
- le rollback est plus difficile si on ne connait pas l'image exacte

Mesures appliquees ou recommandees :

- deployer le tag SHA Git exact, pas seulement `latest`
- proteger la branche `main`
- utiliser un environnement GitHub `production` avec approbation
- limiter les secrets au strict necessaire
- ajouter du scan d'image et, idealement, signer les images

## Front

Le fichier `http-server/index.html` contient un front statique qui consomme :

- `/students`
- `/departments/IRC/students`
- `/departments/ETI/students`
- `/departments/CGP/students`

Il est embarque dans l'image HTTPD et servi directement par Apache.

## Note sur DockerHub

Les images `aymanatics/tp-devops-*:latest` n'existaient pas publiquement au moment du deploiement manuel. Pour finaliser le serveur sans identifiants DockerHub, le playbook a ete lance avec :

```bash
ANSIBLE_HOME=.ansible BUILD_IMAGES_ON_REMOTE=true ansible-playbook playbook.yml
```

Ce mode copie les sources vers `/opt/tp3-app`, build les images sur le serveur, puis lance les memes conteneurs. En CI/CD, le mode normal reste le pull depuis DockerHub apres publication par GitHub Actions.
