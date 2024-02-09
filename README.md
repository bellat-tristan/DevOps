# DevOps
## TP1 
### Question 1-1 :  Document your database container essentials: commands and Dockerfile.
#### DockerFile :
```
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

COPY scripts/01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY scripts/02-InsertData.sql /docker-entrypoint-initdb.d
```
- La première ligne permet d'utiliser l'image postgres:14.1-alpine pour le docker
- Ensuite on définit les variables d'environnement utiles comme les paramètres de connexions à la base de données
  (ici on définit les variables d'environnement POSTGRES_USER et POSGRES_PASSWORD et on leur assigne respectivement les valeurs usr et pwd).
- Les instructions copy permettent de copier des éléments de notre machine hôte sur le docker, ici on fait la copie de scripts sql dans le fichier /docker-entrypoint-initdb. les placer dans ce dossier va nous permettre des exécuter au lancement du docker.

#### Construction du docker :
```
sudo docker build -t Tristan/tp1db .
```
Cette commande va permettre de transformer le DockerFile en image docker.

#### Lancement du docker :
```
sudo docker run -p 5432:5432 --name tp1db --network app-network -v /my/own/datadir:/var/lib/postgresql/data Tristan/tp1db
```
Une fois le build fait on peut lancer le docker via la commande ```docker run```.
Ici on ajoute plusieurs paramètres à la commande :
- ``` -p ``` pour spécifier le port hôte et celui du docker
- ``` --name ``` pour spécifier un nom au conteneur
- ``` --network ``` pour placer notre conteneur dans le même réseaux qu'un outil adminer pour la visualisation de la base de données
- ``` -v ``` pour monter un volumes sur le dockeur afin de ne pas perdre les données de la base de données si on coupe le docker

#### Lancement de l'adminer :
```
sudo docker run     -p "8080:8080"     --net=app-network     --name=adminer     -d     adminer
```
Cette commande permet de lancer un docker qui permettra la visualisation de la base de données grâce à l'outil adminer.
On le place dans le même network que notre base de données pour qu'il puisse accéder aux données.

### Question 1-2 : Why do we need a multistage build ? And explain each step of this dockerfile.

Ici on a besoin d'un build multisatge car on veut pouvoir nommer l'étape de build pour pouvoir l'utiliser dans le run.

#### Dockerfile : 
```
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```
Dans ce Dokerfile on commence par une étape de build :
- On récupère une image maeven qu'on va nommer "myapp-build"
- On définit la variable d'environnement "MYAPP_HOME"
- On change le répertoire de travail
- On fait 2 copie de fichier
- On exécute la commande de build

Ensuite on ajoute une étape de run :
- On récupère une image
- On définit la variable d'environnement "MYAPP_HOME"
- On change le répertoire de travail
- On copie le fichier créer par le build

Enfin on définit l'exécutable par défaut du conteneur.

Commandes pour créer l'image et lancer le docker :
- ```sudo docker build -t Tristan/tp1api .```
- ```sudo docker run -p 8080:8080 --network app-network --name tp1api Tristan/tp1api```

PARTIE HTTP :

Dockerfile :
```
FROM httpd:2.4
COPY ./ /usr/local/apache2/htdocs/
```

```
sudo docker build -t Tristan/tp1http .
```

```
sudo docker run -dit -p 8090:80 --name tp1http  Tristan/tp1http
```

#### GET current conf :
```
sudo docker cp  tp1http:/usr/local/apache2/conf/httpd.conf /tmp/test
```

### Question 1-3 : Why do we need a reverse proxy?

Le reverse proxy va nous servir à n'exposer qu'un seul port sur le réseaux, on a désormais un seul point d'entrée pour notre application.

### Question 1-4 :

#### Fichier : docker-compose.yml
```yaml
version: '3.8'

services:
    backend:
        build:
            ./TP1_api/
        networks:
            - my-network
        depends_on:
            - database
        environment:
            - HOSTNAME=database:5432
            - USER=usr
            - PASSWORD=pwd
            - DB=db
        
    database:
        build:
            ./TP1/
        networks:
            - my-network
        environment:
            - POSTGRES_DB=db
            - POSTGRES_USER=usr
            - POSTGRES_PASSWORD=pwd

    httpd:
        build:
            ./TP1_http/
        ports:
            - "8080:80"
        networks:
            - my-network
        depends_on:
            - backend

networks:
    my-network: 
```

Ce fichier docker-compose.yml permet de ne plus avoir à lancer les conteneurs un par un en se souciant que chacun est la bonne configuration dans son Dockerfile. Maintenant avec ce fichier on écrit les configurations de tous les conteneurs nécessaires dans ce fichier unique qui une fois rédiger permettra avec une seule commande de build tous les conteneurs et de les run.

run command : ```sudo docker-compose up --build```

### Question 1-5 : Document your publication commands and published images in dockerhub.

Pour chaque projet on doit construire l'image du docker puis lui ajouter un tag avec la commande ```docker tag``` pour lui associer une version et enfin on peut publier l'image sur dockerhub via la commande ```docker push```.

#### DB :
- ```sudo docker build -t Tristan/tp1db .```
- ```sudo docker tag Tristan/tp1db tristanbellat/tp1db:1.0```
- ```sudo docker push tristanbellat/tp1db:1.0```

#### BACKEND :
- ```sudo docker build -t Tristan/tp1api .```
- ```sudo docker tag Tristan/tp1pi tristanbellat/tp1api:1.0```
- ```sudo docker push tristanbellat/tp1api```

#### HTTP :
- ```sudo docker build -t Tristan/tp1http .```
- ```sudo docker tag Tristan/tp1http tristanbellat/tp1http:1.0```
- ```sudo docker push tristanbellat/tp1http```

#### Modifications dans le docker compose :
```yaml
version: '3.8'

services:
    backend:
        image: tristanbellat/TP1_api
        networks:
            - my-network
        depends_on:
            - database
        environment:
            - HOSTNAME=database:5432
            - USER=usr
            - PASSWORD=pwd
            - DB=db
        
    database:
        image: tristanbellat/TP1
        networks:
            - my-network
        environment:
            - POSTGRES_DB=db
            - POSTGRES_USER=usr
            - POSTGRES_PASSWORD=pwd

    httpd:
        image: tristanbellat/TP1_http
        ports:
            - "8080:80"
        networks:
            - my-network
        depends_on:
            - backend

networks:
    my-network: 
```
Maintenant que nos images docker sont sûr le dépôt de dockerhub on ne va plus spécifier un build pour avoir une image mais directement aller la chercher sur dockerhub en spécifiant le paramètre image.

## TP2 Discover Github Action

### 2-1 Que sont les conteneurs de test ?

Ce sont simplement des bibliothèques Java qui me permettent d'exécuter un certain nombre de conteneurs Docker pendant les tests. 
Ici, j'utilise le conteneur postgresql pour m'attacher à mon application pendant les tests.

### 2-2 Documentez vos configurations d'actions Github.
#### Nous avons créer des secrets sur github afin de pouvoir se connecter au dockerhub ainsi qu'a sonar:
![Capture d'écran 2024-02-06 17:12:38](https://github.com/bellat-tristan/DevOps/assets/116623829/63f79458-4755-47cb-b5ed-a6883f3d3444)

#### Voici la configuration sans le split du ficher de config
```yaml
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches:                                                 #changement au push des branches main et develop
      -  main
      -  develop

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17       #mise en place de la config java
      - uses: actions/checkout@v4
      - name: Set up JDK 17                                           
        uses: actions/setup-java@v3   
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven                                                                        #utilisation de maven
          
     #finally build your app with the latest command
      - name: Build and test with Maven
        run: |                                                                              #on se rend dans le repertoire ou se trouve le repertoire java
            cd DevOps/TP1_api/
            mvn -B verify sonar:sonar -Dsonar.projectKey=bellat-tristan_DevOps -Dsonar.organization=bellat-tristan -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml                                       #verifation et dialogue avec sonar

  # define job to build and publish docker image
  build-and-push-docker-image:
   needs: test-backend
   # run only when code is compiling and tests are passing
   runs-on: ubuntu-22.04
  
   # steps to perform in job
   steps:
     - name: Login to DockerHub                                                                     #connection a dockerhub
       run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}      #utilisation des secrets gihub pour la connection

     - name: Checkout code
       uses: actions/checkout@v2.5.0
  
     - name: Build image and push backend                                                            #generation et push de l'image du docker backend
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./DevOps/TP1_api/                                                               #chemin du dockerfile
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1api:latest                                       #tag de l'image docker
         push: ${{ github.ref == 'refs/heads/main' }}                                               #push de l'image docker si dans la branche main
         
     - name: Build image and push database                                                            #generation et push de l'image du docker base de données
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./DevOps/TP1/                                                               #chemin du dockerfile
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1:latest                                       #tag de l'image docker
         push: ${{ github.ref == 'refs/heads/main' }}                                               #push de l'image docker si dans la branche main

     - name: Build image and push httpd                                                            #generation et push de l'image du docker serveur web
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./DevOps/TP1_http/                                                               #chemin du dockerfile
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1_http:latest                                       #tag de l'image docker
         push: ${{ github.ref == 'refs/heads/main' }}                                               #push de l'image docker si dans la branche main
```

#### Voici la configuration une fois le split effectué du publish_docker
```yaml
name: publish_docker
on:
  workflow_run:
    workflows: [test_backend]
    types:
      - completed
    branches:
      - main
jobs:
 # define job to build and publish docker image
  build-and-push-docker-image:
   # run only when code is compiling and tests are passing
   runs-on: ubuntu-22.04
   if: ${{ github.event.workflow_run.conclusion == 'success'}}   #Ici on attend que le premier pipeline soit terminer avec un success
  
   # steps to perform in job
   steps:
     - name: Login to DockerHub                                                                  #connection au dockerhub
       run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}   #recuperation des secrets pour connection

     - name: Checkout code
       uses: actions/checkout@v2.5.0
  
     - name: Build image and push backend
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./DevOps/TP1_api/                                                            #chemin du dockfile
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1api:latest                                   #tag de l'image du docker
         push: ${{ github.ref == 'refs/heads/main' }}                                           #push du docker de l'image si on est sur la branche main
         
     - name: Build image and push database
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./DevOps/TP1/                                                            #chemin du dockfile
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1:latest                                   #tag de l'image du docker
         push: ${{ github.ref == 'refs/heads/main' }}                                           #push du docker de l'image si on est sur la branche main

     - name: Build image and push httpd
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./DevOps/TP1_http/                                                            #chemin du dockfile
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1_http:latest                                   #tag de l'image du docker
         push: ${{ github.ref == 'refs/heads/main' }}                                           #push du docker de l'image si on est sur la branche main
      
```
#### Voici la configuration une fois le split effectuer du test_backend
```yaml
name: test_backend
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      -  main
      -  develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - uses: actions/checkout@v4
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
          
     #finally build your app with the latest command
      - name: Build and test with Maven
        run: |
            cd DevOps/TP1_api/         #chemin du repertoire java
            mvn -B verify sonar:sonar -Dsonar.projectKey=bellat-tristan_DevOps -Dsonar.organization=bellat-tristan -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml   #Verification et echange avec sonar
```
### Documentez la configuration de votre portail.
#### Creation du Token pour la connection :
![Capture d'écran 2024-02-06 17:18:30](https://github.com/bellat-tristan/DevOps/assets/116623829/1d82423a-b128-48ef-b347-de57093cefc2)
#### Ficher de configuration test_backend modifié pour etres utilisé avec sonar:
```yaml
run: |
   cd DevOps/TP1_api/         #chemin du repertoire java
   mvn -B verify sonar:sonar -Dsonar.projectKey=bellat-tristan_DevOps -Dsonar.organization=bellat-tristan -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml   #Verification et echange avec sonar
```

#### Visualisation des information concerant la "verifiaction" de sonar :
![Capture d'écran 2024-02-06 17:21:04](https://github.com/bellat-tristan/DevOps/assets/116623829/2a30f63c-36b9-419c-87ee-4d184d96496f)

## TP3
### 3-1 Documentez votre inventaire et vos commandes de base
#### Voici les commandes de base pour ansible :
![Capture d'écran 2024-02-07 09:53:25](https://github.com/bellat-tristan/DevOps/assets/116623829/46c16d29-61e7-43d6-883c-d257f7397c6c)

#### Voici la configuration de l'inventaire ansible du projet:
```yaml
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: ./id_rsa
 children:
   prod:
     hosts: centos@tristan.bellat.takima.cloud
```
### 3-2 Documentez votre playbook
#### Configuration du premier playbook:
```yaml
- hosts: all
  gather_facts: false
  become: true

  tasks:
   - name: Test connection
     ping:
```
#### Execution du premier playbook.yml:
![Capture d'écran 2024-02-07 10:10:09](https://github.com/bellat-tristan/DevOps/assets/116623829/73f78140-0af0-456f-b973-9713317c288a)

#### Configuration du deuxieme playboob.yml:
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
#### Execution du deusieme playbook.yml:
![Capture d'écran 2024-02-07 10:13:33](https://github.com/bellat-tristan/DevOps/assets/116623829/f8183bfb-0806-4f20-923b-e30fc970e4b3)

#### Verification de l'installation de docker sur le serveur:
![Capture d'écran 2024-02-07 10:15:46](https://github.com/bellat-tristan/DevOps/assets/116623829/081fd7f5-65bd-4a98-aa49-74d9653be9a3)
##### Creation du roles/docker:
```ansible-galaxy init roles/docker```
![Capture d'écran 2024-02-07 10:55:16](https://github.com/bellat-tristan/DevOps/assets/116623829/4d50bac1-d22e-42de-b886-bcf0dc75ceeb)

##### Modification du playbook (creation du roles/docker) :
```yaml
- hosts: all
  gather_facts: false
  become: true

# Install Docker
  tasks:
  
  - name: Role/docker
    include_role: 
      name: roles/docker
```
#### Modification du main.yml dans le roles/docker/task pour installer docker:
```yaml
---
# tasks file for roles/docker

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
#### Exucution du playbook avec le roles/docker
![Capture d'écran 2024-02-07 11:16:35](https://github.com/bellat-tristan/DevOps/assets/116623829/0e30a251-c707-434f-894a-8de2fd2160d2)
### Documentez la configuration de vos tâches docker_container.
#### fichier de configuration task database:
```yaml
# tasks file for roles/database
  - name: Run database
    docker_container:
      name: database
      image: tristanbellat/tp1
      networks:
          - name: my-network
```
#### fichier de configuration task api:
```yaml
  - name: backend
    docker_container:
      name: backend
      image: tristanbellat/tp1api
      networks:
          - name: my-network
```
#### fichier de configuration task proxy:
```yaml
  - name: Run HTTPD
    docker_container:
      name: httpd
      image: tristanbellat/tp1http
      ports:
          - "80:80"
      networks:
          - name: my-network
```
#### fichier de configuration task docker:
```yaml
# tasks file for roles/docker

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
#### fichier de configuration task network:
```yaml
# tasks file for roles/network
  - name: Run network
    docker_network:
      name: my-network
      state: present
```
#### Fichier de configuration du front:
```yaml
---
# tasks file for roles/front
  - name: Run Front
    docker_container:
      name: front
      image: tristanbellat/tp3front:latest
      networks:
          - name: my-network
      pull: true
```
#### Modification du fichier playbook pour ajout du front:
```yaml
- hosts: all
  gather_facts: false
  become: true
  vars_files: 
    - vault.yml

# Install Docker
  tasks:
  
  - name: Role/docker
    include_role: 
      name: roles/docker
  
# Install network

  - name: Role/network
    include_role:
      name: roles/network

# Install database

  - name: Role/database
    include_role:
      name: roles/database 

# Install proxy

  - name: Role/proxy
    include_role:
      name: roles/proxy

# Install api

  - name: Role/api
    include_role:
      name: roles/api

# Install front
  - name: Role/front
    include_role:
      name: roles/front
```
#### fichier de setup modifier pour les variables d'environement:
```yaml
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: ./id_rsa
   HOSTNAME: database:5432
   USER: usr
   PASSWORD: pwd
   DB: db
   POSTGRES_DB: "db"
   POSTGRES_USER: "usr"
   POSTGRES_PASSWORD: "pwd"
 children:
   prod:
     hosts: centos@tristan.bellat.takima.cloud
```
#### fichier de connection a la base de donnée:
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
    url: jdbc:postgresql://${HOSTNAME}/${DB}
    username: ${USER}
    password: ${PASSWORD}
    driver-class-name: org.postgresql.Driver
management:
 server:
   add-application-context-header: false
 endpoints:
   web:
     exposure:
       include: health,info,env,metrics,beans,configprops
```
### Deployer automatiquement le projet :
#### Creation d'un secret pour la clé SHH:
![image](https://github.com/bellat-tristan/DevOps/assets/116623829/fdbf55e9-a2f8-4b8f-bc89-22241e3de448)

#### Modification du fichier d'environement:
```VUE_APP_API_URL=tristan.bellat.takima.cloud/api```
#### Modification du fichier de configuration httpd pour redirection:
```yaml
ServerName tristan.bellat.takima.cloud
<VirtualHost *:80>
ProxyPreserveHost On

ProxyPass /api/ http://backend:8080/
ProxyPassReverse /api/ http://backend:8080/

ProxyPass / http://front:80/
ProxyPassReverse / http://front:80/

</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

#### Fichier playbook modifier pour l'ajout de Vault :
Vault permet de chiffrer les données sensible du projet (mot de passe de la base de données ...)
Chiffrement de Vault: AES256
```yaml
- hosts: all
  gather_facts: false
  become: true
  vars_files: 
    - vault.yml

# Install Docker
  tasks:
  
  - name: Role/docker
    include_role: 
      name: roles/docker
  
# Install network

  - name: Role/network
    include_role:
      name: roles/network

# Install database

  - name: Role/database
    include_role:
      name: roles/database 

# Install proxy

  - name: Role/proxy
    include_role:
      name: roles/proxy

# Install api

  - name: Role/api
    include_role:
      name: roles/api

# Install front
  - name: Role/front
    include_role:
      name: roles/front
```
#### Fichier Vault :
```yaml
$ANSIBLE_VAULT;1.1;AES256
37636431636632303131313133393432313531653064383265303139626661623333616137393134
6663666635363765393732373831326362663463626132630a613733336635656533656538323533
30613062306533393135353933323433613262316331643236653939653630643666353364636334
3037346532303831330a323563386663626162363863336639396436336663626235306134393433
32336134393432666466643132386433356132363365613238653839303666333732616264313861
64383338373536316633303033353238356362366438376363383430393831633039383231303836
65343031313061643966363737396335326139653234336536633133613930663961623166333037
37303764633236613934393135313933303335396231373930346433663431383433616661623136
38643762383961646237393634653637323930376565613739653630616639396531623937373037
31336531396265386631353733373033373962613337616366613833303031316161393139373932
62376337353336303263383335636331336164316264383666323664646536653164623838316565
64326432613464333836363363633233626331353136376463623037623363643966643837363935
36336634613562653563333038643234303330323361323432333661356334343430333737383364
66303536346465623338643836333136343665643238363437306334646366646461383634663234
34336430383637313365333136613031376439333065386538656339623161306332623939663963
64303566316332623736643233343761666636383763616134343932373637356166376232633838
33666133383436626662663237383662646365343730313464653865373364666330653066323132
3065333361393262336438363237343234663863333131343931
```
#### Creation d'un secret pour le fichier Vault:
![image](https://github.com/bellat-tristan/DevOps/assets/116623829/9fbf51fc-132d-4127-8ac6-830d883d1ae4)

#### Fichier de deploiement automatique avec Vault:
le deploiement se fait quand il y a un push sur la branche main.
```yaml
name: deploy
on:
  #to begin you want to launch this job in main and develop
  workflow_run:
    workflows: ["publish_docker"]
    types:
      - completed
    branches: 
    - main

jobs:
  # define job to build and publish docker image
  deploy:
   # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04
    # Environment for the variables
    environment: production
    # only run job if the previous job was successful
    if: github.event.workflow_run.conclusion == 'success'   # steps to perform in job
    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0
      
      - name: Install ansible
        run: sudo apt-get install ansible
        
      - name: Setting up Vault password
        run: |
          echo "${{ secrets.VAULT_PASSWORD }}" > ~/vault_pass.pem
          chmod 600 ~/vault_pass.pem
          
      - name: Deploy to server with ansible
        working-directory: ./DevOps/TP3/ansible/
        run: |
            mkdir -p ~/.ssh
            echo "${{secrets.SSH_PRIV_KEY_DEVOPS}}" > ~/.ssh/ssh_priv_key_devops
            chmod 400 ~/.ssh/ssh_priv_key_devops
            ssh-keyscan -H tristan.bellat.takima.cloud >> ~/.ssh/known_hosts
            ansible-playbook -i inventories/setup.yml playbook.yml --vault-password-file=~/vault_pass.pem
```
#### Modification du fichier de setup (Vault):
```yaml
all:
 vars:
   ansible_user: centos
   docker_network_name: my-network
 children:
   prod:
     hosts: centos@tristan.bellat.takima.cloud
```

