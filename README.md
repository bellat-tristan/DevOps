# DevOps
## TP1
### 1-1 Documentez les éléments essentiels de votre conteneur de base de données : commandes et Dockerfile.
#### Dockerfile :
```
FROM postgres:14.1-alpine                                            //j'utilise l'image postgres:14.1-alpine  

ENV POSTGRES_DB=db \                                                 //  je defini les variables d'environements
   POSTGRES_USER=usr \                                               
   POSTGRES_PASSWORD=pwd


COPY scripts/01-CreateScheme.sql /docker-entrypoint-initdb.d        // Permet de copy les srcipt SQL dans le repertoire du docker /docker-entrypoint-initdb.d 
COPY scripts/02-InsertData.sql /docker-entrypoint-initdb.d          //afin qu'il puisse genrer les tables des bases de données au lancement de celui-ci
   ```
#### les scripts SQL
Les scripts permttent de créer les tables dans le base ainsi que de les remplir.
##### 01-CreateScheme.sql
```
CREATE TABLE public.departments
(
 id      SERIAL      PRIMARY KEY,
 name    VARCHAR(20) NOT NULL
);

CREATE TABLE public.students
(
 id              SERIAL      PRIMARY KEY,
 department_id   INT         NOT NULL REFERENCES departments (id),
 first_name      VARCHAR(20) NOT NULL,
 last_name       VARCHAR(20) NOT NULL
);
```
##### 02-InsertData.sql
```
INSERT INTO departments (name) VALUES ('IRC');
INSERT INTO departments (name) VALUES ('ETI');
INSERT INTO departments (name) VALUES ('CGP');


INSERT INTO students (department_id, first_name, last_name) VALUES (1, 'Eli', 'Copter');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Emma', 'Carena');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Jack', 'Uzzi');
INSERT INTO students (department_id, first_name, last_name) VALUES (3, 'Aude', 'Javel');
```
#### La commande pour Build le Docker :
Elle permet de build le Dockerfile en image docker
```
sudo docker build -t Tristan/tp1db .
```
#### La commande pour créer le network:
Permet de connecter le Docker au adminer
```
sudo docker network create app-network
```
#### La commande pour lancer le Docker:
cette commande permet de lancer le docker (tp1db) en lui affectant un volume de stockage pour ne pas perdre les données ajouter/modifier de la base de données
elle permet egalement de le connceter au network et de lui affecter des ports.
```
sudo docker run -v /my/own/datadir:/var/lib/postgresql/data --net=app-network -p 5432:5432 --name tp1db Tristan/tp1db
```
#### La commande pour lancer le adminer:
cette commande permet de lancer le adminer, de le connceter au network et de lui affecter des ports.
```
sudo docker run -p "8080:8080" --net=app-network --name=adminer -d adminer
```
### API
#### 1-2 Why do we need a multistage build? And explain each step of this dockerfile.
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
### Docker compose
```
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

### 
```
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

