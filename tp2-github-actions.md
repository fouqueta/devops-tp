# TP 2 - GitHub Actions

# First steps into the CI World

## Build and test your Application

- Pour build et run les tests :

`mvn clean verify -f backend/pom.xml`

- Extrait du pom.xml du backend :

```xml
<dependencies>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>${testcontainers.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>jdbc</artifactId>
        <version>${testcontainers.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <version>${testcontainers.version}</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

→ Les Testcontainers sont des bibliothèques Java de tests qui fournissent des API faciles et légères pour initialiser des tests d’intégration avec de véritables services encapsulés dans des conteneurs Docker. Cela permet d’écrire des tests qui interagissent avec le même type de services que ceux utilisés en production, sans avoir recours à des simulations ou à des services en mémoire.

- Création du main.yml

```yaml
name: CI devops 2024
on:
  push:
    branches: 
      - main
      - develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v2.5.0

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
            java-version: '17'
            distribution: 'temurin'
            cache: maven

      - name: Build and test with Maven
        run: mvn clean verify -f backend/pom.xml
```

- `name: CI devops 2024` : définit le nom du workflow d’intégration continue (CI)
- `on:
  push:
    branches: 
      - main
      - develop` : spécifie les événements qui déclencheront le workflow. Ici, le workflow se déclenche lorsqu’on commit sur les branches `main` et `develop`.
- `jobs:
  test-backend: 
    runs-on: ubuntu-22.04` : définit un job appelé `test-backend` qui s’exécute sur une machine virtuelle Ubuntu 22.04
- `steps:
  - uses: actions/checkout@v2.5.0` : première étape du job `test-backend`. Elle utilise l’action `actions/checkout@v2.5.0` pour récupérer le code de notre dépôt.
- `- name: Set up JDK 17 
   uses: actions/setup-java@v3 
   with: 
     java-version: '17' 
     distribution: 'temurin' 
     cache: maven` : deuxième étape du job `test-backend`. Elle utilise l’action `actions/setup-java@v3` pour configurer le JDK 17 avec la distribution temurin. Elle met également en cache les dépendances Maven pour accélérer les prochains buildss.
- `- name: Build and test with Maven 
   run: mvn clean verify -f backend/pom.xml` : dernière étape du job `test-backend`. Elle exécute la commande `mvn clean verify -f backend/pom.xml` pour construire et tester votre application.

En résumé, ce workflow récupère notre code, configure l’environnement Java, construit et teste notre application chaque fois qu’un push est effectué sur les branches `main` et `develop`.

## First steps into the CD World

1. Ajouter les secrets DOCKERHUB_PWD et DOCKERHUB_USERNAME dans github
2. Ajouter  dans le fichier main.yml un job pour build nos images docker lors de la pipeline :

```yaml
name: CI devops 2024
on:
  push:
    branches: 
      - main
      - develop
  pull_request:

jobs:  
  # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04
  
    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0
  
      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./backend
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-backend:latest
  
      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-database:latest
  
      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./httpd
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-httpd:latest
```

- `needs: test-backend` :  signifie que le job `build-and-push-docker-image` ne s’exécutera que si le job `test-backend` s’est terminé avec succès. C’est une façon de s’assurer que l’on ne construit et ne pousse des images Docker que lorsque notre code compile et que tous les tests passent.
1. Modifier le fichier main.yml pour publier les images docker quand il y a un commit sur la branche main :

```yaml
jobs:  
  # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04
  
    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0
  
		  # on se connecte à notre compte DockerHub grâce à nos secrets
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_PWD }}
     
      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./backend
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-backend:latest
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}
      
      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}
  
      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./httpd
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-httpd:latest
          push: ${{ github.ref == 'refs/heads/main' }}
```

# Setup Quality Gate

## Register to SonarCloud

- Créer un compte sur SonarCloud
- Créer un secret SONAR_TOKEN dans github correspondant au token qu’on a créé dans sonarcloud
- Modifier le fichier main.yml pour lancer l’analyse SonarCloud dans l’étape Build and test with Maven du job test-backend :

```yaml
jobs:
  # define job to build and test backend
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v2.5.0

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      - name: Build and test with Maven
        run: mvn -B clean verify sonar:sonar -Dsonar.projectKey=fouqueta_devops-tp -Dsonar.organization=devops-tp-2024 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} -f backend/pom.xml  
```

L’option `-B` dans la commande `mvn -B verify` est un raccourci pour **`-**-batch-mode`. Cela permet d’exécuter Maven en mode non interactif. C’est particulièrement utile lors de l’exécution de Maven dans des environnements automatisés, comme les pipelines d’intégration continue (CI), où l’interaction de l’utilisateur n’est pas possible.

On passe en paramètre la clé du projet et la clé de l’organisation qu’on l’on peut retrouver dans sonarcloud.

On doit aussi renseigner le login sonar avec la valeur du secret SONAR_TOKEN que l’on a définit en précédemment.

# Bonus: split pipelines (Optional)

Séparation des 2 jobs en 2 workflows différents pour que :

- `test-backend` soit lancé sur develop et main et `build-and-push-docker-image` sur main seulement
- `build-and-push-docker-image` soit lancé seulement si `test-backend` passe avec succès

Le fichier test-backend.yml :

```yaml
name: Test Backend
on:
  push:
    branches: 
      - main
      - develop
  pull_request:
    branches:
      - main
      - develop

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v2.5.0
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
      
      - name: Build and test with Maven
        run: mvn -B clean verify sonar:sonar -Dsonar.projectKey=fouqueta_devops-tp -Dsonar.organization=devops-tp-2024 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} -f backend
```

Le fichier build-and-push-docker-image.yml :

```yaml
name: Build and Push Docker Image
on:
  workflow_run:
    workflows: ["Test Backend"]
    types:
      - completed

jobs:
  build-and-push-docker-image:
    runs-on: ubuntu-22.04
    if: ${{ github.ref == 'refs/heads/main' && github.event.workflow_run.conclusion == 'success' }}
    steps:
      - uses: actions/checkout@v2.5.0
      
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_PWD }}
        
      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          context: ./backend
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-backend:latest
          push: true
          
      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-database:latest
          push: true
          
      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./httpd
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-httpd:latest
          push: true
```