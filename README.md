# TP : Découvrez Docker

## Description des dossiers et fichiers
- **backend/** : contient une première application Java simple, ainsi que son Dockerfile, utilisée pour les premières étapes du TP sur la conteneurisation d'une application.
- **database/** : contient le Dockerfile de l'image de base de données PostgreSQL et les scripts SQL d'initialisation du schéma et d'insertion des données.
- **flask-app/** : contient une application Flask réalisée dans le cadre d'un TD distinct du TP Docker principal.
- **http-server/** : contient les fichiers nécessaires au serveur HTTP Apache,notamment son Dockerfile, sa configuration et sa page HTML. Apache est également configuré pour servir de reverse proxy vers l'API backend.
- **simpleapi/** : contient l'API Java Spring Boot utilisée pour le TP avec son Dockerfile, son fichier pom.xml et son code source dans src/.

## Réponses aux questions 

### 1-1 Pour quelle raison vaut-il mieux lancer le conteneur avec un drapeau pour donner les variables d’environnement plutôt que de les placer directement dans le fichier Docker ?

Définir les variables d'environnement au lancement du conteneur permet de séparer la configuration de l'application de la construction de son image. Cette méthode présente plusieurs avantages :
- **Sécurité :** évite d'inscrire directement des identifiants dans le Dockerfile et dans l'image construite.
- **Flexibilité :** permet d'utiliser des configurations différentes selon l'environnement (développement, test ou production).
- **Réutilisabilité :** une même image peut être utilisée avec plusieurs configurations sans être reconstruite.

### 1-2 Pourquoi avons-nous besoin d’un volume à attacher à notre conteneur de postgres ?

Par défaut, les données de PostgreSQL sont stockées dans le système de fichiers du conteneur. Si celui-ci est supprimé, les données qu'il contient peuvent être perdues. Un volume Docker permet de conserver les données indépendamment du cycle de vie du conteneur. Elles restent disponibles après sa suppression et lors de la création d'un nouveau conteneur utilisant le même volume.

### 1-3 Éléments essentiels du conteneur de base de données : commandes et Dockerfile

#### Commandes essentielles

Construire l'image :

```bash
docker build -t milaepf/database ./database
```

Créer le réseau de communication :

```bash
docker network create app-network
```

Créer un volume persistant :

```bash
docker volume create milaepf-db-data
```

Lancer la base de données :

```bash
docker run -d \
  --name milaepf-db \
  --network app-network \
  -e POSTGRES_DB=db \
  -e POSTGRES_USER=usr \
  -e POSTGRES_PASSWORD=pwd \
  -v milaepf-db-data:/var/lib/postgresql/data \
  milaepf/database
```

Lancer Adminer sur le même réseau :

```bash
docker run -d \
  --name adminer \
  --network app-network \
  -p 8090:8080 \
  adminer
```

Adminer permet d'administrer la base de données depuis un navigateur, à l'adresse `http://localhost:8090`.

### 1-4 Pourquoi avons-nous besoin d'une construction multi-étapes ? Expliquez chaque étape du Dockerfile.

Une construction multi-étapes permet de séparer la compilation d'une application de son exécution. Elle permet notamment de :
- Réduire la taille de l'image finale en n'y conservant que les éléments nécessaires à l'exécution.
- Ne pas installer Maven et le JDK dans l'image finale.
- Séparer les outils de compilation de l'environnement d'exécution.
- Limiter les composants inutiles présents dans l'image finale.

#### Dockerfile utilisé

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME

RUN apk add --no-cache maven

COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run stage
FROM eclipse-temurin:21-jre-alpine
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

#### Explication des instructions

| Instruction | Rôle |
|---|---|
| `FROM eclipse-temurin:21-jdk-alpine AS myapp-build` | Crée la première étape avec le JDK 21, nécessaire pour compiler le code Java. L'étape est nommée `myapp-build`. |
| `ENV MYAPP_HOME=/opt/myapp` | Définit une variable d'environnement correspondant au répertoire de travail de l'application |
| `WORKDIR $MYAPP_HOME` | Définit le répertoire de travail dans le conteneur |
| `RUN apk add --no-cache maven` | Installe Maven, nécessaire pour construire le projet |
| `COPY pom.xml .` | Copie le fichier de configuration Maven dans l'image |
| `COPY src ./src` | Copie les sources Java du projet |
| `RUN mvn package -DskipTests` | Compile l'application et génère le fichier JAR sans exécuter les tests |
| `FROM eclipse-temurin:21-jre-alpine` | Démarre une seconde étape, plus légère, contenant le JRE 21 nécessaire à l'exécution |
| `ENV MYAPP_HOME=/opt/myapp` | Redéfinit la variable dans la seconde étape, indépendante de la première |
| `WORKDIR $MYAPP_HOME` | Définit le répertoire de travail de l'image finale |
| `COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar` | Copie le JAR généré lors de la première étape vers l'image finale |
| `ENTRYPOINT ["java", "-jar", "myapp.jar"]` | Lance l'application Java lorsque le conteneur démarre |

La première étape utilise un JDK et Maven pour construire l'application. La seconde utilise uniquement un JRE pour exécuter le JAR obtenu.

### 1-5 Pourquoi avons-nous besoin d'un reverse proxy ?

Un reverse proxy est un serveur intermédiaire qui reçoit les requêtes des clients et les transmet aux serveurs applicatifs appropriés. Dans notre projet, Apache HTTP Server reçoit les requêtes HTTP et les transmet à l'API Spring Boot.

Les principaux avantages sont :
- **Point d'entrée unique :** les utilisateurs accèdent à l'application via le serveur HTTP.
- **Masquage du backend :** le port de l'API n'a pas besoin d'être exposé directement sur la machine hôte.
- **Sécurité :** il est possible de centraliser certaines règles d'accès et la terminaison TLS.
- **Évolutivité :** un reverse proxy peut répartir les requêtes entre plusieurs serveurs applicatifs.
- **Centralisation :** il peut faciliter la gestion des accès et de certaines configurations HTTP.

### 1-6 Pourquoi docker-compose est-il si important ?

Docker Compose permet de définir et d'orchestrer plusieurs conteneurs à partir d'un seul fichier YAML. Dans notre projet, il coordonne les trois services : le serveur HTTP Apache, l'API Spring Boot et la base de données PostgreSQL.

Il facilite :
- Le démarrage et l'arrêt de l'ensemble des services
- La construction des images
- La configuration des réseaux et des volumes
- La définition des variables d'environnement
- L'organisation des dépendances entre services
- La reproductibilité de l'environnement sur différentes machines

Sans Docker Compose, il faudrait gérer séparément les commandes de construction et de lancement de chaque conteneur.

### 1-7 Documenter le docker - composer les commandes les plus importantes.

Les commandes suivantes permettent de gérer l'application depuis le répertoire contenant `docker-compose.yml`.

| Commande | Fonction |
|---|---|
| `docker compose config` | Vérifie et affiche la configuration Compose résolue |
| `docker compose build` | Construit les images des services |
| `docker compose up -d` | Crée et démarre les services en arrière-plan |
| `docker compose up -d --build` | Reconstruit les images puis démarre les services |
| `docker compose ps` | Affiche l'état des services et des conteneurs. |
| `docker compose logs` | Affiche les journaux des services |
| `docker compose logs -f` | Suit les journaux en temps réel |
| `docker compose logs -f backend` | Suit uniquement les journaux du backend |
| `docker compose stop` | Arrête les conteneurs sans les supprimer |
| `docker compose start` | Redémarre les conteneurs arrêtés |
| `docker compose restart` | Redémarre les services |
| `docker compose down` | Arrête et supprime les conteneurs et les réseaux créés par Compose |

### 1-8 Documentez votre fichier docker-compose.

Le fichier `docker-compose.yml` décrit les services, leurs configurations et les ressources nécessaires à leur fonctionnement.

#### Principales propriétés

| Propriété | Fonction |
|---|---|
| `services` | Définit les conteneurs de l'application |
| `build` | Indique le contexte utilisé pour construire l'image d'un service |
| `context` | Définit le chemin du dossier contenant les fichiers nécessaires à la construction |
| `environment` | Définit les variables d'environnement du conteneur |
| `networks` | Associe les services aux réseaux Docker |
| `ports` | Publie des ports du conteneur sur la machine hôte |
| `volumes` | Permet de conserver les données en dehors du cycle de vie d'un conteneur |
| `depends_on` | Définit les dépendances entre services et peut conditionner le démarrage à un contrôle de santé |
| `healthcheck` | Vérifie l'état de santé d'un service |
| `restart` | Définit la politique de redémarrage d'un conteneur |
| `internal: true` | Crée un réseau interne qui limite les communications vers l'extérieur |

#### Organisation des réseaux

Le projet utilise deux réseaux :
- `frontend-net` : permet au serveur HTTP de communiquer avec le backend.
- `backend-net` : permet au backend de communiquer avec PostgreSQL.

Le backend appartient aux deux réseaux, tandis que la base de données est uniquement connectée à `backend-net`. Le serveur HTTP est uniquement connecté à `frontend-net`. Cette séparation limite les communications directes entre les services et évite d'exposer inutilement la base de données.


### 1-9 Documentez vos commandes de publication et vos images publiées dans dockerhub.

Docker Hub est un registre en ligne qui permet de stocker et de partager des images Docker.

#### Étape 1 : se connecter à Docker Hub

```bash
docker login
```
Cette commande permet de s'authentifier auprès du registre.

#### Étape 2 : taguer les images

Le tag associe une image locale à un nom de dépôt et à une version sur Docker Hub.

```bash
docker tag milaepf/database milaepf2/database:1.0
docker tag milaepf/simpleapi milaepf2/simpleapi:1.0
docker tag milaepf/http-server milaepf2/http-server:1.0
```

#### Étape 3 : publier les images

```bash
docker push milaepf2/database:1.0
docker push milaepf2/simpleapi:1.0
docker push milaepf2/http-server:1.0
```

Une image publiée peut être récupérée avec docker pull, par exemple :

```bash
docker pull milaepf2/database:1.0
```

### 1-10 Pourquoi mettons-nous nos images dans un dépôt en ligne ?

La publication des images dans un registre en ligne permet de les partager et de les réutiliser sur différentes machines sans devoir reconstruire systématiquement les images à partir du code source. Elle présente plusieurs avantages :
- **Partage** : les membres d'une équipe peuvent récupérer les mêmes images.
- **Déploiement** : les images peuvent être téléchargées sur un serveur distant.
- **Reproductibilité** : une image publiée correspond à un environnement construit et identifié.
- **Versionnement** : les tags permettent de distinguer différentes versions d'une application.
- **Gain de temps** : il n'est pas nécessaire de reconstruire l'image sur chaque machine qui l'utilise.

