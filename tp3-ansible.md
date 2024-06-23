# TP 3 - Ansible

# Introduction

## Inventories

- Créer un fichier /ansible/inventories/setup.yml :

```yaml
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: /home/aurelie/id_rsa
  children:
    prod:
      hosts: aurfouquet.takima.cloud
```

- J’ai ajouté le nom de mon serveur [aurfouquet.takima.cloud](http://aurfouquet.takima.cloud) à la variable `hosts`, et le chemin de ma clé privé /home/aurelie/id_rsa à la variable `ansible_ssh_private_key_file`.
- Lancer la commande `ansible all -i inventories/setup.yml -m ping` me renvoie bien pong.

## Facts

- On demande à notre serveur de nous dire sur quelle distribution on est grâce au mode setup :

`ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"`

Ce qui nous donne : 

```yaml
aurfouquet.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "CentOS",
        "ansible_distribution_file_parsed": true,
        "ansible_distribution_file_path": "/etc/redhat-release",
        "ansible_distribution_file_variety": "RedHat",
        "ansible_distribution_major_version": "7",
        "ansible_distribution_release": "Core",
        "ansible_distribution_version": "7.9",
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}
```

On est donc sur une distribution CentOS.

- On retire le serveur Apache httpd que l’on a installé dans le TD :

`ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become`

# **Playbooks**

## First playbook

- Créer un fichier /ansible/playbook.yml :

```yaml
- hosts: all
  gather_facts: false
  become: true

  tasks:
    - name: Test connection
      ping:
```

- On vérifie que notre playbook est correct :

`ansible-playbook --syntax-check -i inventories/setup.yml playbook.yml`

Cette commande vérifie la syntaxe de notre playbook en utilisant l’inventaire spécifié dans `inventories/setup.yml`. Si la syntaxe de notre playbook est correcte, la commande ne renvoie aucune sortie. Si notre playbook contient des erreurs de syntaxe, la commande affiche un message d’erreur.

- On exécute notre playbook avec la commande :

`ansible-playbook -i inventories/setup.yml playbook.yml`

## **Advanced Playbook**

- Modifier le fichier playbook.yml pour installer docker sur notre serveur :

```yaml
- hosts: all
  gather_facts: false
  become: true

  # Install Docker
  tasks:

  - name: Install device-mapper-persistent-data
    yum:
      name: device-mapper-persistent-data
      state: latest

  - name: Install lvm2
    yum:
      name: lvm2
      state: latest

  - name: add repo docker
    command:
      cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install Docker
    yum:
      name: docker-ce
      state: present

  - name: Install python3
    yum:
      name: python3
      state: present

  - name: Install docker with Python 3
    pip:
      name: docker
      executable: pip3
    vars:
      ansible_python_interpreter: /usr/bin/python3

  - name: Make sure Docker is running
    service: name=docker state=started
    tags: docker
```

`Install device-mapper-persistent-data` : installe le paquet `device-mapper-persistent-data` qui est une dépendance pour Docker

`Install lvm2` : installe le paquet `lvm2` qui est également une dépendance pour Docker

`add repo docker` : ajoute le dépôt Docker à la liste des dépôts yum, ce qui permet d’installer Docker à partir de ce dépôt

`Install Docker` :  installe Docker à partir du dépôt que l’on a ajouté précédemment

`Install python3` :  installe Python 3, qui est nécessaire pour certaines fonctionnalités de Docker

`Install docker with Python 3` :  installe le module Docker pour Python 3. Cela permet à Ansible de gérer Docker à l’aide de scripts Python

`Make sure Docker is running` :  s’assure que le service Docker est en cours d’exécution. Si Docker n’est pas en cours d’exécution, cette tâche le démarrera

- On exécute notre playbook avec `ansible-playbook -i inventories/setup.yml playbook.yml`
- Pour vérifier que Docker est bien installé et en cours d’exécution, on peut exécuter les commandes suivantes :

→ depuis notre PC : `ansible all -i inventories/setup.yml -a "docker ps" --become` 

→ depuis le serveur (on se connecte avec `ssh -i id_rsa centos@aurfouquet.takima.cloud`) : `sudo docker ps`

## **Using roles**

- Tout d’abord, on crée un rôle docker avec la commande suivante :

`ansible-galaxy init roles/docker`

- Cela nous crée de nouveaux dossiers dans roles/docker. On garde seulement les dossiers tasks et handlers.
- On modifie le fichier /ansible/roles/docker/tasks/main.yml qui a été généré, en y mettant la configuration de la tâche de l’installation docker :

```yaml
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: Add Docker repo
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present

- name: Install python3
  yum:
    name: python3
    state: present

- name: Install Docker Python module
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3

- name: Ensure Docker is running
  service: name=docker state=started
```

- On modifie le fichier /ansible/playbook.yml pour gérer le nouveau rôle docker :

```yaml
- hosts: all
  gather_facts: false
  become: true

  roles:
    - docker
```

# Deploy your App

On crée un rôle spécifique pour chaque partie de notre application :

- `ansible-galaxy init roles/install-docker`
- `ansible-galaxy init roles/create-network`
- `ansible-galaxy init roles/launch-database`
- `ansible-galaxy init roles/launch-app`
- `ansible-galaxy init roles/launch-proxy`

## `install-docker`

Pour le rôle `install-docker`, cela ne change pas de la partie précédente :

```yaml
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: Add Docker repo
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present

- name: Install python3
  yum:
    name: python3
    state: present

- name: Install Docker Python module
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3

- name: Ensure Docker is running
  service: name=docker state=started
```

## `create-network`

Pour le rôle `create-network`, le main.yml est le suivant :

```yaml
- name: Create network
  docker_network:
    name: my-network
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

On utilise le module `docker_network` pour créer un réseau Docker. Lorsque l’on crée le réseau, on doit s’assurer que le bon interpréteur Python est utilisé. On peut le faire en définissant la variable `ansible_python_interpreter` à /usr/bin/python3 . Cela permet de créer le réseau `my-network`. 

## `launch-database`

Pour le rôle `launch-database`, le main.yml est le suivant :

```yaml
- name: Launch database
  docker_container:
    name: tp-database-1
    image: fouqueta/tp-database:latest
    state: started
    recreate: yes
    volumes:
      - /data
    env:
      POSTGRES_USER: usr
      POSTGRES_PASSWORD: pwd
      POSTGRES_DB: db
    networks:
      - name: my-network
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

Ce rôle lance un conteneur Docker pour notre base de données. 

Il utilise l’image `fouqueta/tp-database:latest`, et nomme le conteneur `tp-database-1`. 

Le conteneur est configuré pour démarrer automatiquement (`state: started`), et pour être recréé si nécessaire (`recreate: yes`). 

Un volume est créé à `/data` pour stocker les données de la base de données. 

Les variables d’environnement sont définies pour configurer la base de données PostgreSQL. 

Le conteneur est connecté au réseau `my-network`.

## `launch-app`

Pour le rôle `launch-app`, le main.yml est le suivant :

```yaml
- name: Launch app
  docker_container:
    name: tp-backend-1
    image: fouqueta/tp-backend:latest
    state: started
    recreate: yes
    env:
      SPRING_DATASOURCE_URL: jdbc:postgresql://tp-database-1:5432/db
    networks:
      - name: my-network
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

Ce rôle lance un conteneur Docker pour notre application backend. 

Il utilise l’image `fouqueta/tp-backend:latest`, et nomme le conteneur `tp-backend-1`. 

Le conteneur est également configuré pour démarrer automatiquement et être recréé si nécessaire. 

Une variable d’environnement est définie pour configurer l’URL de la source de données de l’application, qui pointe vers le container de la base de données que nous avons lancée précédemment.

Le conteneur est également connecté au réseau `my-network`.

(Le nom du container est bien le même nom qui est configuré dans le fichier httpd.conf qui est copié dans l’image du proxy.)

## `launch-proxy`

Pour le rôle `launch-proxy`, le main.yml est le suivant :

```yaml
- name: Launch proxy
  docker_container:
    name: tp-httpd-1
    image: fouqueta/tp-httpd:latest
    state: started
    recreate: yes
    ports:
      # Publish container port 80 as host port 8080
      - "8080:80"
    networks:
      - name: my-network
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

Ce rôle lance un conteneur Docker pour notre serveur proxy. 

Il utilise l’image `fouqueta/tp-httpd:latest`, et nomme le conteneur `tp-httpd-1`. 

Le conteneur est également configuré pour démarrer automatiquement et être recréé si nécessaire. 

Le port 80 du conteneur est mappé au port 8080 de l’hôte, ce qui signifie que l’on peut accéder au serveur proxy en accédant au port 8080 de notre machine hôte. 

Le conteneur est également connecté au réseau `my-network`.

## Playbook.yml

On ajoute tous les rôles dans le playbook :

```yaml
- hosts: all
  gather_facts: false
  become: true

  roles:
    - install-docker
    - create-network
    - launch-database
    - launch-app
    - launch-proxy
```

- On exécute notre playbook avec `ansible-playbook -i inventories/setup.yml playbook.yml`
- On peut tester que notre déploiement fonctionne en allant sur [http://aurfouquet.takima.cloud:8080/departments/IRC/students](http://aurfouquet.takima.cloud:8080/departments/IRC/students)

# Front

- J’ai modifié le fichier build-and-push-docker-image.yml pour rajouter la publication de l’image frontend :

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
          
      - name: Build image and push frontend
        uses: docker/build-push-action@v3
        with:
          context: ./frontend
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-frontend:latest
          push: true
```

- J’ai modifié le httpd.conf pour gérer le frontend :

```vbnet
ServerRoot "/usr/local/apache2"
ServerName localhost

<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass /api http://tp-backend-1:8080/
ProxyPassReverse /api http://tp-backend-1:8080/
ProxyPass / http://tp-frontend-1:8081/
ProxyPassReverse / http://tp-frontend-1:8081/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

- J’ai modifié les fichiers .env.development et .env.production pour mettre le chemin de l’api :

```vbnet
VUE_APP_API_URL=/api
```

- J’ai créé un nouveau rôle launch-frontend pour déployer le container frontend, et rajouté ce rôle dans le playbook.yml :

```yaml
- name: Launch frontend
  docker_container:
    name: tp-frontend-1
    image: fouqueta/tp-frontend:latest
    state: started
    recreate: yes
    ports:
      - "8081:80"
    networks:
      - name: my-network
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

```yaml
- hosts: all
  gather_facts: false
  become: true

  roles:
    - install-docker
    - create-network
    - launch-database
    - launch-app
    - launch-proxy
    - launch-frontend
```

Cependant, après déploiement des containers sur le server, je n’ai pas réussi à accéder au frontend sur [http://aurfouquet.takima.cloud:8081/](http://aurfouquet.takima.cloud:8081/).