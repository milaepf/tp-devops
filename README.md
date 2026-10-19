# TP : Découvrez Docker

## Introduction
L'objectif de ce TP est de conteneuriser une application web composée de trois niveaux : une base de données PostgreSQL, un backend API développé en Java avec Spring Boot et un serveur HTTP Apache. Les composants sont ensuite regroupés avec Docker Compose puis les images sont publiées sur Docker Hub.

## 1. Base de données
L'image PostgreSQL postgres:17.2-alpine est utilisée pour créer la base de données. Un Dockerfile est créé dans le dossier database, puis l'image est construite avec :

docker build -t milaepf/database .

Les variables d'environnement permettent de configurer la base db, l'utilisateur usr et son mot de passe.
Deux scripts SQL sont ajoutés dans database/init/ : 01-CreateScheme.sql crée les tables departments et students, tandis que 02-InsertData.sql insère les départements IRC, ETI et CGP ainsi que quatre étudiants.
Les scripts sont copiés dans /docker-entrypoint-initdb.d/ pour être exécutés lors de la première initialisation d'une base vide. Adminer est utilisé pour vérifier les tables et les données.
Enfin, un volume Docker nommé milaepf-db-data est monté dans /var/lib/postgresql/data afin de conserver les données même si le conteneur est supprimé et recréé.

## 2. Backend API
Une première application Java affichant Hello World! est compilée puis exécutée dans un conteneur utilisant un environnement JRE.
Le backend est ensuite développé avec Spring Boot et Maven. Le Dockerfile utilise une construction multi-étape : la première image, basée sur eclipse-temurin:21-jdk-alpine, compile le projet, et la seconde, basée sur eclipse-temurin:21-jre-alpine, exécute uniquement le fichier JAR généré.
L'image est construite avec :

docker build -t milaepf/simpleapi .

Le backend est ensuite connecté à PostgreSQL grâce à une configuration JDBC. Avec Docker Compose, l'adresse utilisée est jdbc:postgresql://database:5432/db, où database correspond au nom du service Docker.
Le point de terminaison /departments/IRC/students permet de vérifier que l'API récupère les étudiants enregistrés dans la base de données.

## 3. Serveur HTTP Apache
Le serveur HTTP utilise l'image httpd:2.4-alpine. Le dossier http-server contient un Dockerfile, une page d'accueil index.html et la configuration httpd.conf.
L'image est construite avec :

docker build -t milaepf/http-server .

Apache est configuré comme un reverse proxy grâce aux modules mod_proxy et mod_proxy_http. Les directives ProxyPass et ProxyPassReverse transmettent les requêtes au backend.
Dans la configuration finale, Apache utilise http://backend:8080/ comme destination. Le serveur HTTP devient ainsi le point d'entrée de l'application.

## 4. Orchestration avec Docker Compose
Le fichier compose.yaml, placé à la racine du projet, décrit les trois services : database, backend et httpd.
Deux réseaux sont utilisés :
- frontend-net relie Apache au backend ;
- backend-net relie le backend à PostgreSQL.

Le volume milaepf-db-data assure la persistance des données. Seul le port du serveur HTTP est exposé sur la machine hôte. Les ports du backend et de la base de données restent internes au réseau Docker.
La configuration est vérifiée puis l'application est démarrée avec :
- docker compose config
- docker compose up -d –build
- docker compose ps

Les journaux peuvent être consultés avec docker compose logs. L'application est ensuite accessible via http://localhost/, et le point de terminaison /departments/IRC/students permet de tester la communication entre les trois niveaux.

## 5. Publication sur Docker Hub
Les images sont d'abord vérifiées avec docker images. Après connexion au compte Docker Hub avec docker login, un tag 1.0 est ajouté à chaque image.
- docker tag milaepf/database milaepf2/database:1.0
- docker tag milaepf/simpleapi milaepf2/simpleapi:1.0
- docker tag milaepf/http-server milaepf2/http-server:1.0

Les images sont ensuite publiées avec :
- docker push milaepf2/database:1.0
- docker push milaepf2/simpleapi:1.0
- docker push milaepf2/http-server:1.0

La publication est vérifiée dans les dépôts du compte. Les images peuvent ensuite être téléchargées depuis une autre machine avec docker pull.

## Conclusion
Ce TP m'a permis de découvrir la construction et l'exécution d'images Docker, la persistance des données avec les volumes et la communication entre conteneurs grâce aux réseaux. J'ai également appris à utiliser une construction multi-étapes pour le backend Java, à configurer Apache comme reverse proxy et à orchestrer plusieurs services avec Docker Compose. Enfin, la publication sur Docker Hub permet de partager et de réutiliser les images sur différentes machines.

