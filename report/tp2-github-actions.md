# TP2 - GitHub Actions

## Etat actuel

- Le TP a ete finalise avec des workflows separes dans `.github/workflows/`.
- `backend-ci.yml` execute `test-backend` sur `main` et `develop`.
- `quality-gate.yml` est declenche apres succes de `Backend CI`.
- `docker-publish.yml` est declenche apres succes de `Backend CI`, mais seulement pour `main`.

## Etape 1 - CI backend

Le premier objectif du TP est de tester automatiquement l'application a chaque `push` sur `main` et `develop`, ainsi qu'a chaque `pull_request`.

Commande utilisee dans le workflow :

```bash
mvn -B verify --file ./simple-api-student-main/pom.xml
```

Pourquoi `verify` ?

- `clean` supprime les artefacts precedents.
- `test` lance les tests unitaires.
- `verify` va plus loin et declenche aussi les tests d'integration via Failsafe.

## Etape 2 - Testcontainers

### 2-1 Que sont les Testcontainers ?

Les Testcontainers sont des bibliotheques Java qui demarrent des conteneurs Docker temporaires pendant les tests.

Dans ce projet :

- les tests d'integration utilisent PostgreSQL via `jdbc:tc:postgresql`
- la base de test est creee a la demande
- l'environnement de test est proche de la realite

Important :

- les tests IT ne peuvent pas passer si Docker n'est pas demarre
- dans ce workspace, l'erreur observee est `Could not find a valid Docker environment`

## Etape 3 - Variables securisees GitHub

Secrets a ajouter dans GitHub :

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
- `SONAR_TOKEN`

Variables GitHub a ajouter :

- `SONAR_PROJECT_KEY`
- `SONAR_ORGANIZATION`

### 2-2 Pourquoi utiliser des variables securisees ?

Les variables securisees servent a proteger les informations sensibles :

- elles evitent d'exposer des mots de passe et tokens dans le depot
- elles permettent d'utiliser les memes workflows sur plusieurs environnements
- elles limitent le risque de fuite dans l'historique Git

## Etape 4 - Build et publication des images Docker

Les images construites sont :

- backend : `simple-api-student-main`
- database : `./`
- http server : `http-server`

Le workflow final est separe :

- `Backend CI` verifie le code sur `develop` et `main`
- `Build And Push Docker Images` ne se declenche que si `Backend CI` a reussi sur `main`

### 2-3 Pourquoi utiliser `needs: test-backend` ?

`needs: test-backend` garantit que la phase Docker ne demarre que si la compilation et les tests passent.

Sans `needs` :

- les jobs partent en parallele
- on peut publier des images issues d'un code casse
- on perd l'interet de la CI comme garde-fou

### 2-4 Pourquoi pousser des images Docker ?

Pousser les images vers un registre permet :

- de reutiliser une image deja construite sans rebuilder le code
- de deployer plus vite sur d'autres machines
- de versionner et partager les artefacts produits par la CI

## Etape 5 - Quality Gate SonarCloud

Le workflow `Quality Gate` est configure pour se lancer apres succes de `Backend CI`.

Il te reste a :

1. creer un compte SonarCloud
2. creer une organisation
3. recuperer `SONAR_PROJECT_KEY`
4. recuperer `SONAR_ORGANIZATION`
5. ajouter `SONAR_TOKEN` dans les secrets GitHub

## Etape 6 - Going further: split pipelines

La derniere partie du TP demandait de separer les pipelines.

Implementation retenue :

- `backend-ci.yml` : tests backend sur `develop` et `main`
- `docker-publish.yml` : publication Docker uniquement apres succes de `Backend CI` sur `main`
- `quality-gate.yml` : analyse SonarCloud apres succes de `Backend CI`

Pourquoi utiliser `workflow_run` ?

- cela permet de declencher un workflow uniquement si un autre est termine
- on peut filtrer sur le succes du workflow precedent
- on evite de publier des images si les tests ont echoue

## Etape 7 - Verification a faire sur ta machine / GitHub

1. demarrer Docker Desktop ou un daemon Docker compatible
2. pousser ce projet dans un vrai depot GitHub
3. ajouter les secrets/variables dans `Settings > Secrets and variables > Actions`
4. pousser sur `develop` pour verifier le job `test-backend`
5. pousser sur `main` pour verifier le push des images Docker
6. verifier le rapport SonarCloud

## Limites de la verification locale dans ce workspace

Je n'ai pas pu valider le pipeline de bout en bout ici pour deux raisons externes au code :

- le dossier courant n'est pas initialise en depot Git
- Docker n'est pas disponible dans cet environnement, donc Testcontainers echoue localement
