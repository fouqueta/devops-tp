# TP 1 - Docker

# Database

## Basics

- Pour build l’image postgres :

`docker build -t my_postgres .`

- Pour run le container  :

`docker run --name my_postgres -p 5432:5432 -d my_postgres`

- Re-run la base de données avec adminer :

`docker run --name my_postgres --network app-network -p 5432:5432 -d my_postgres`

`docker run --name my_adminer --network app-network -p 8080:8080 -d adminer`

- Pour run un container avec des paramètres d’environnement :

`docker run --name my_postgres -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd -p 5432:5432 -d my_postgres`

## Init database

- Pour créer un network :

`docker network create app-network`

- Pour run l’adminer :

`docker run -p "8090:8080" --net=app-network --name=my_adminer -d adminer`

- Le nouveau Dockerfile pour initialiser et peupler notre base de données :

```docker
# utiliser une image postgres
FROM postgres:14.1-alpine

# copier les scripts de sql-scripts dans /docker-entrypoint-initdb.d du container
COPY ./sql-scripts /docker-entrypoint-initdb.d
```

- Pour run le container :

`docker run --network=app-network -p 5432:5432 -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd --name=my_postgres -d my_postgres`

- Pour vérifier si notre base de données a des données, on peut se connecter à [localhost:8090](http://localhost:8090) grâce à l’adminer qui tourne à cette adresse. 
Sur l’interface, sélectionner :
→ système : PostgreSQL
→ serveur : le nom du container de notre base de données, soit `my_postgres`
→ utilisateur : usr
→ mot de passe : pwd
→ base de données : db

## **Persist data**

- Pour persister les données :

`docker run --network=app-network -p 5432:5432 -v myPsqlVolume:/var/lib/postgresql/data -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd --name=my_postgres -d my_postgres`

- Pour être sûr que les données sont bien persistées :
→ se connecter à la base avec l’adminer sur [localhost:8090](http://localhost:8090)
→ insérer un nouvel étudiant `(5, 1, 'Alain', 'Térieur')` par exemple
→ tuer le container `my_postgres` avec `docker rm -f <id>` 
→ rerun le container avec `docker run ...`
→ avec l’adminer, voir que l’on a toujours Alain Térieur dans la table students

# Backend API

## Basics

- Compiler notre classe :

`javac Main.java`

- Le Dockerfile :

```docker
# Use an official Java Runtime Environment (JRE) base image
FROM eclipse-temurin:17-jre-alpine

# Set the working directory in the container
WORKDIR /app

# Copy the compiled Java class file into the container
COPY Main.class /app

# Run the Java application when the container starts
CMD ["java", "Main"]
```

- Build l’image :

`docker build -t my-app .`

- Run le container :

`docker run -it --name my-app my-app`

## **Multistage build**

Une construction multi-étapes permet de séparer les dépendances de construction des dépendances d’exécution. On peut ainsi garder notre image finale petite et efficace, contenant seulement ce qui est nécessaire pour exécuter l’application.

- Le Dockerfile :

```docker
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

- Etape de build

`FROM maven:3.8.6-amazoncorretto-17 AS myapp-build` : spécifie l’image de base pour l’étape de construction, utilise une image Maven avec le jdk Amazon Corretto 17

`ENV MYAPP_HOME /opt/myapp` : définit une variable d’environnement `MYAPP_HOME` qui pointe vers le répertoire `/opt/myapp`

`WORKDIR $MYAPP_HOME` : définit le répertoire de travail dans l’image Docker à la valeur de `MYAPP_HOME`

`COPY pom.xml .` : copie le fichier `pom.xml` vers l’image Docker

`RUN mvn dependency:go-offline` : exécute la commande Maven pour télécharger toutes les dépendances nécessaires pour le projet et les mettre en cache

`COPY src ./src` : copie le répertoire `src` vers l’image Docker

`RUN mvn package -DskipTests` : exécute la commande Maven pour construire l’application, en sautant les tests

- Etape de run

`FROM amazoncorretto:17` : spécifie l’image de base pour l’étape d’exécution, utilise une image Amazon Corretto 17 jre

`ENV MYAPP_HOME /opt/myapp` : définit une variable d’environnement `MYAPP_HOME` qui pointe vers le répertoire `/opt/myapp`

`WORKDIR $MYAPP_HOME` : définit le répertoire de travail dans l’image Docker à la valeur de `MYAPP_HOME`

`COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar` : copie le fichier jar construit de l’étape de construction dans l’image

`ENTRYPOINT java -jar myapp.jar` : définit le point d’entrée pour l’image Docker, qui est la commande qui sera exécutée lorsqu’un conteneur est démarré à partir de l’image. La commande exécute notre application java.

- Le fichier de configuration application.yml du projet java :

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    generate-ddl: false
    open-in-view: true
  datasource:
    url: jdbc:postgresql://my_postgres:5432/db
    username: usr
    password: pwd
    driver-class-name: org.postgresql.Driver
management:
 server:
   add-application-context-header: false
 endpoints:
   web:
     exposure:
       include: health,info,env,metrics,beans,configprops

```

- Build l’image :

`docker build -t my-app .`

- Run le container :

`docker run --net=app-network -it -p 8080:8080 --name my-app-app my-app`

- Tester l’appli en allant sur  [http://localhost:8080/departments/IRC/students](http://localhost:8080/departments/IRC/students)

# Http server

## Basics

```docker
# utiliser une image httpd
FROM httpd:2.4

# copier l'index.html dans le dossier du container
COPY ./index.html /usr/local/apache2/htdocs/
```

- Pour build l’image :

`docker build -t my-http-server .`

- Pour run le container :

`docker run --net=app-network -dit -p 8080:80 --name my-http-server my-http-server`

- Tester que tout marche  en allant sur [http://localhost:8080/](http://localhost:8080/) : on doit voir la page correspondant au index.html

## Configuration

- Avec docker exec

`docker exec my-http-server cat /usr/local/apache2/conf/httpd.conf > httpd.conf`

- avec docker cp

`docker cp my-http-server:/usr/local/apache2/conf/httpd.conf .`

## **Reverse proxy**

- Modifier le fichier de configuration httpd.conf :

```vbnet
ServerRoot "/usr/local/apache2"
ServerName localhost

<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://my-app:8080/
ProxyPassReverse / http://my-app:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

- Copier le fichier httpd.conf avec les nouveaux changements dans le container :

```docker
# utiliser une image httpd
FROM httpd:2.4

# copier l'index.html dans le dossier du container
COPY ./index.html /usr/local/apache2/htdocs/
# copier le fichier de configuration dans le dossier du container
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```

- Pour tester l’appli avec le reverse proxy :

→ d’abord lancer notre container postgres my-postgres

→ puis run le container du backend my-app sans exposer les ports (sans l’option -p 8080:8080)

→ enfin lancer le container du reverse proxy my-http-server en exposant les ports (-p 8080:80)

→ aller sur [http://localhost:8080/departments/IRC/students](http://localhost:8080/departments/IRC/students)

# **Link application**

## **Docker-compose**

- Créer le fichier docker-compose.yml :

```yaml
version: '3.7'

services:
    backend:
        build: ./backend
        networks:
          - my-network
        depends_on:
          - database

    database:
        build: ./database
        networks:
          - my-network
        environment:
          POSTGRES_PASSWORD: pwd

    httpd:
        build: ./httpd
        ports:
          - "8080:80"
        networks:
          - my-network
        depends_on:
          - backend

networks:
    my-network:
```

- Les containers s’appellent maintenant tp-database-1, tp-backend-1 et tp-httpd-1, et les images tp-database, tp-backend et tp-httpd (noms donnés par défaut  quand je fais `docker-compose up`)
- Il faut donc changer les noms de containers dans les urls du httpd.conf et de l’application.yml avec les nouveaux noms de containers

→ on accède à notre API via [http://localhost:8180](http://localhost:8080)

Explications du fichier docker-compose.yml :

- `services` : définit les services (c’est-à-dire les conteneurs) à créer
    - `backend`, `database`, **`httpd`** : noms des services
    - `build` : chemin vers le Dockerfile que docker compose doit utiliser pour construire l’image du service. Dans notre cas, docker compose s’attend à trouver les Dockerfiles dans les répertoires `./backend`, `./database` et `./httpd`
    - `networks` ****: permet de connecter le service à un ou plusieurs réseaux. Dans notre cas, tous les services sont connectés au réseau `my-network`
    - `depends_on` ****: indique que le service dépend d’un autre service. Par exemple, le service `backend` ****dépend du service `database`, ce qui signifie que le service `database` sera démarré avant le service `backend`
    - `ports` : permet de mapper les ports du conteneur aux ports de l’hôte. Dans notre cas, le port 80 du conteneur `httpd` est mappé au port 8080 de l’hôte
- `networks` : définit les réseaux que l’on veut. Dans notre cas, on crée le réseau my-network

Commandes docker-compose courantes :

`docker-compose up` : construit, crée, démarre et attache les conteneurs pour un service

`docker-compose down` : arrête et supprime les conteneurs, les réseaux, les images et les volumes

`docker-compose build` : construit ou reconstruit les services

`docker-compose start` : démarre les services existants

`docker-compose stop` : arrête les services

`docker-compose restart` : redémarre les services

`docker-compose pull` : pull les images de service

`docker-compose logs` : affiche les logs des services

# Publish

- Pour se connecter à Docker Hub :

`docker login`

On doit rentrer notre nom d’utilisateur et notre mot de passe Docker Hub.

- Pour tagger les images :

`docker tag tp-database fouqueta/tp-database:1.0`

`docker tag tp-backend fouqueta/tp-backend:1.0`

`docker tag tp-httpd fouqueta/tp-httpd:1.0`

fouqueta est mon USERNAME et 1.0 est le numéro de version de mes images

- Pour push les images sur Docker Hub

`docker push fouqueta/tp-database:1.0`

`docker push fouqueta/tp-backend:1.0`

`docker push fouqueta/tp-httpd:1.0`

Ces commandes envoie nos images au dépôt Docker Hub associé à mon nom d’utilisateur. Une fois l’opération terminée, nos images seront disponibles pour être téléchargées par n’importe qui ayant accès à Docker Hub.