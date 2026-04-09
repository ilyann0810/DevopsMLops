# TP Part 01 - Docker

## Goals

### Good practice
- Documenter chaque étape (le rapport sera évalué)
- Créer une structure de dossiers appropriée : 1 dossier par image

### Target application
Application 3-tiers :
- HTTP server
- Backend API
- Database

---

## 1. Database

### 1.1 Basics

Image utilisée : `postgres:17.2-alpine`

**Dockerfile minimal :**
```dockerfile
FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd
```

**Commandes :**
```bash
# Créer le network
docker network create app-network

# Build l'image
docker build -t my-database ./database

# Run le container
docker run -d --name my-database --network app-network my-database

# Run adminer pour visualiser la DB
docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
```

> **Question 1-1** : Why is it better to use `-e` flag for environment variables rather than putting them in the Dockerfile?

**Réponse :** Utiliser le flag `-e` est préférable pour plusieurs raisons :
1. **Sécurité** : Les mots de passe et données sensibles ne sont pas stockés en dur dans le Dockerfile, qui peut être versionné sur Git et donc exposé.
2. **Flexibilité** : On peut changer les valeurs sans reconstruire l'image (différentes configs pour dev/prod).
3. **Réutilisabilité** : La même image peut être utilisée dans différents environnements avec des configurations différentes.
4. **Bonnes pratiques** : Les secrets ne doivent jamais être committés dans le code source.

---

### 1.2 Init database

Créer les scripts SQL dans un dossier `database/` :

**01-CreateScheme.sql**
```sql
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

**02-InsertData.sql**
```sql
INSERT INTO departments (name) VALUES ('IRC');
INSERT INTO departments (name) VALUES ('ETI');
INSERT INTO departments (name) VALUES ('CGP');

INSERT INTO students (department_id, first_name, last_name) VALUES (1, 'Eli', 'Copter');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Emma', 'Carena');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Jack', 'Uzzi');
INSERT INTO students (department_id, first_name, last_name) VALUES (3, 'Aude', 'Javel');
```

**Dockerfile mis à jour :**
```dockerfile
FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY *.sql /docker-entrypoint-initdb.d/
```

---

### 1.3 Persist data

Utiliser un volume pour persister les données :

```bash
docker run -d \
    --name my-database \
    --network app-network \
    -v /my/own/datadir:/var/lib/postgresql/data \
    my-database
```

> **Question 1-2** : Why do we need a volume to be attached to our postgres container?

**Réponse :** Un volume est nécessaire car :
1. **Persistance des données** : Sans volume, toutes les données sont perdues quand le container est supprimé (les containers sont éphémères).
2. **Survie aux redémarrages** : Les données persistent même si on rebuild l'image ou recrée le container.
3. **Séparation des préoccupations** : Les données sont séparées du container, facilitant les backups et migrations.
4. **Performance** : Les volumes Docker sont optimisés pour les I/O par rapport au système de fichiers du container.

---

> **Question 1-3** : Document your database container essentials: commands and Dockerfile.

**Réponse :**

**Dockerfile :**
```dockerfile
FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY *.sql /docker-entrypoint-initdb.d/
```

**Commandes essentielles :**
```bash
# Créer le réseau
docker network create app-network

# Construire l'image
docker build -t my-database ./database

# Lancer le container avec volume
docker run -d \
    --name my-database \
    --network app-network \
    -v db-data:/var/lib/postgresql/data \
    -e POSTGRES_DB=db \
    -e POSTGRES_USER=usr \
    -e POSTGRES_PASSWORD=pwd \
    my-database

# Vérifier que le container tourne
docker ps

# Voir les logs
docker logs my-database
```

---

## 2. Backend API

### 2.1 Basics - Hello World

**Main.java**
```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello World!");
    }
}
```

**Commandes :**
```bash
# Compiler
javac Main.java

# Dockerfile basique
FROM eclipse-temurin:21-jre-alpine
COPY Main.class .
CMD ["java", "Main"]
```

---

### 2.2 Multistage build

**Dockerfile multistage :**
```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /usr/src
COPY Main.java .
RUN javac Main.java

# Run stage
FROM eclipse-temurin:21-jre-alpine
COPY --from=build /usr/src/Main.class .
CMD ["java", "Main"]
```

---

### 2.3 Backend Simple API (Spring Boot)

Créer une application Spring Boot sur [Spring Initializer](https://start.spring.io/) :
- Project: Maven
- Language: Java 21
- Spring Boot: 3.4.5
- Packaging: Jar
- Dependencies: Spring Web

**GreetingController.java**
```java
package fr.takima.training.simpleapi.controller;

import org.springframework.web.bind.annotation.*;
import java.util.concurrent.atomic.AtomicLong;

@RestController
public class GreetingController {

    private static final String template = "Hello, %s!";
    private final AtomicLong counter = new AtomicLong();

    @GetMapping("/")
    public Greeting greeting(@RequestParam(value = "name", defaultValue = "World") String name) {
        return new Greeting(counter.incrementAndGet(), String.format(template, name));
    }

    record Greeting(long id, String content) {}
}
```

**Dockerfile :**
```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
RUN apk add --no-cache maven
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Run stage
FROM eclipse-temurin:21-jre-alpine
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

> **Question 1-4** : Why do we need a multistage build? And explain each step of this dockerfile.

**Réponse :**

**Pourquoi un multistage build ?**
1. **Réduction de la taille de l'image** : L'image finale ne contient que le JRE et le JAR, pas le JDK ni Maven (économie de plusieurs centaines de Mo).
2. **Sécurité** : Moins d'outils dans l'image finale = moins de surface d'attaque.
3. **Autonomie** : Pas besoin d'avoir Maven/JDK installé sur la machine hôte pour builder.

**Explication de chaque étape :**

```dockerfile
# STAGE 1 : Build
FROM eclipse-temurin:21-jdk-alpine AS myapp-build  # Image avec JDK pour compiler
ENV MYAPP_HOME=/opt/myapp                          # Variable d'environnement pour le chemin
WORKDIR $MYAPP_HOME                                # Définit le répertoire de travail
RUN apk add --no-cache maven                       # Installe Maven
COPY pom.xml .                                     # Copie le fichier de dépendances
RUN mvn dependency:go-offline                      # Télécharge les dépendances (cache)
COPY src ./src                                     # Copie le code source
RUN mvn package -DskipTests                        # Compile et crée le JAR

# STAGE 2 : Run
FROM eclipse-temurin:21-jre-alpine                 # Image légère avec JRE uniquement
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar  # Copie le JAR depuis le stage 1
ENTRYPOINT ["java", "-jar", "myapp.jar"]           # Lance l'application
```

---

### 2.4 Backend API (avec connexion DB)

Télécharger le code source : [simple-api](https://github.com/takima-training/simple-api)

Configurer `application.yml` pour la connexion à la base de données.

**Commandes :**
```bash
docker build -t my-backend ./backend

docker run -d \
    --name my-backend \
    --network app-network \
    -p 8080:8080 \
    my-backend
```

Tester : `http://localhost:8080/departments/IRC/students`

---

## 3. HTTP Server

### 3.1 Basics

**Dockerfile :**
```dockerfile
FROM httpd:2.4

COPY index.html /usr/local/apache2/htdocs/
```

**index.html**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Welcome</title>
</head>
<body>
    <h1>Hello from Apache!</h1>
</body>
</html>
```

---

### 3.2 Configuration

Récupérer la config par défaut :
```bash
docker exec -it my-httpd cat /usr/local/apache2/conf/httpd.conf > httpd.conf
```

---

### 3.3 Reverse Proxy

Ajouter à `httpd.conf` :
```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://backend:8080/
    ProxyPassReverse / http://backend:8080/
</VirtualHost>
```

**Dockerfile :**
```dockerfile
FROM httpd:2.4

COPY httpd.conf /usr/local/apache2/conf/httpd.conf
```

> **Question 1-5** : Why do we need a reverse proxy?

**Réponse :** Un reverse proxy est nécessaire pour :
1. **Point d'entrée unique** : Un seul port exposé (80/443) pour accéder à toute l'application.
2. **Sécurité** : Le backend n'est pas directement exposé à Internet, seul le proxy est accessible.
3. **SSL/TLS** : Gestion centralisée des certificats HTTPS.
4. **Load balancing** : Distribution du trafic entre plusieurs instances backend.
5. **Cache** : Mise en cache des ressources statiques.
6. **Servir le frontend** : Le même serveur peut servir l'application frontend (HTML/JS/CSS) et router les appels API vers le backend.

---

## 4. Docker Compose

### 4.1 docker-compose.yml

```yaml
version: '3.8'

services:
  database:
    build: ./database
    container_name: my-database
    networks:
      - app-network
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd

  backend:
    build: ./backend
    container_name: my-backend
    networks:
      - app-network
    depends_on:
      - database
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://database:5432/db
      - SPRING_DATASOURCE_USERNAME=usr
      - SPRING_DATASOURCE_PASSWORD=pwd

  httpd:
    build: ./httpd
    container_name: my-httpd
    ports:
      - "80:80"
    networks:
      - app-network
    depends_on:
      - backend

networks:
  app-network:

volumes:
  db-data:
```

### 4.2 Commandes docker-compose

```bash
# Démarrer tous les services
docker compose up -d

# Voir les logs
docker compose logs -f

# Arrêter tous les services
docker compose down

# Rebuild et redémarrer
docker compose up -d --build

# Supprimer tout (y compris volumes)
docker compose down -v
```

> **Question 1-6** : Why is docker-compose so important?

**Réponse :** Docker-compose est important car :
1. **Orchestration simplifiée** : Gère plusieurs containers avec une seule commande.
2. **Configuration déclarative** : Tout est défini dans un fichier YAML versionnable.
3. **Reproductibilité** : Même environnement pour tous les développeurs et en production.
4. **Gestion des dépendances** : `depends_on` assure l'ordre de démarrage des services.
5. **Réseau automatique** : Les services peuvent communiquer entre eux par leur nom.
6. **Volumes partagés** : Gestion centralisée des volumes persistants.
7. **Variables d'environnement** : Configuration facile sans modifier les images.

---

> **Question 1-7** : Document docker-compose most important commands.

**Réponse :**

| Commande | Description |
|----------|-------------|
| `docker compose up` | Démarre tous les services |
| `docker compose up -d` | Démarre en arrière-plan (detached) |
| `docker compose up --build` | Rebuild les images avant de démarrer |
| `docker compose down` | Arrête et supprime les containers |
| `docker compose down -v` | Supprime aussi les volumes |
| `docker compose ps` | Liste les services en cours |
| `docker compose logs` | Affiche les logs de tous les services |
| `docker compose logs -f backend` | Suit les logs d'un service spécifique |
| `docker compose exec backend sh` | Ouvre un shell dans un container |
| `docker compose restart` | Redémarre tous les services |
| `docker compose pull` | Télécharge les dernières images |

---

> **Question 1-8** : Document your docker-compose file.

**Réponse :**

```yaml
version: '3.8'

services:
  # Service de base de données PostgreSQL
  database:
    build: ./database              # Construit l'image depuis ./database/Dockerfile
    container_name: my-database    # Nom du container
    networks:
      - app-network                # Connecté au réseau app-network
    volumes:
      - db-data:/var/lib/postgresql/data  # Volume pour persister les données
    environment:                   # Variables d'environnement (credentials)
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd

  # Service backend Spring Boot
  backend:
    build: ./backend               # Construit l'image depuis ./backend/Dockerfile
    container_name: my-backend
    networks:
      - app-network
    depends_on:
      - database                   # Attend que database soit démarré
    environment:                   # Configuration de connexion à la DB
      - SPRING_DATASOURCE_URL=jdbc:postgresql://database:5432/db
      - SPRING_DATASOURCE_USERNAME=usr
      - SPRING_DATASOURCE_PASSWORD=pwd

  # Service HTTP (reverse proxy Apache)
  httpd:
    build: ./httpd                 # Construit l'image depuis ./httpd/Dockerfile
    container_name: my-httpd
    ports:
      - "80:80"                    # Expose le port 80 sur l'hôte
    networks:
      - app-network
    depends_on:
      - backend                    # Attend que backend soit démarré

# Définition du réseau
networks:
  app-network:                     # Réseau bridge pour la communication inter-containers

# Définition des volumes
volumes:
  db-data:                         # Volume nommé pour la persistance des données PostgreSQL
```

---

## 5. Publish

### 5.1 Publication sur Docker Hub

```bash
# Se connecter
docker login

# Taguer les images
docker tag my-database USERNAME/my-database:1.0
docker tag my-backend USERNAME/my-backend:1.0
docker tag my-httpd USERNAME/my-httpd:1.0

# Pusher les images
docker push USERNAME/my-database:1.0
docker push USERNAME/my-backend:1.0
docker push USERNAME/my-httpd:1.0
```

> **Question 1-9** : Document your publication commands and published images in dockerhub.

**Réponse :**

**Commandes de publication :**
```bash
# 1. Se connecter à Docker Hub
docker login

# 2. Taguer les images avec le format USERNAME/IMAGE:TAG
docker tag my-database USERNAME/my-database:1.0
docker tag my-backend USERNAME/my-backend:1.0
docker tag my-httpd USERNAME/my-httpd:1.0

# 3. Pousser les images vers Docker Hub
docker push USERNAME/my-database:1.0
docker push USERNAME/my-backend:1.0
docker push USERNAME/my-httpd:1.0
```

**Images publiées :**
| Image | Tag | Description |
|-------|-----|-------------|
| USERNAME/my-database | 1.0 | PostgreSQL 17.2 avec schéma initialisé |
| USERNAME/my-backend | 1.0 | API Spring Boot Java 21 |
| USERNAME/my-httpd | 1.0 | Apache HTTP Server avec reverse proxy |

---

> **Question 1-10** : Why do we put our images into an online repo?

**Réponse :** On publie nos images sur un registry en ligne pour :
1. **Partage** : Les autres membres de l'équipe peuvent utiliser les mêmes images.
2. **Déploiement** : Les serveurs de production peuvent pull les images directement.
3. **CI/CD** : Les pipelines d'intégration continue peuvent accéder aux images.
4. **Versioning** : Historique des versions avec les tags (1.0, 1.1, latest...).
5. **Backup** : Les images sont sauvegardées en dehors de la machine locale.
6. **Accessibilité** : Disponible depuis n'importe quelle machine avec Docker.

---

## Structure de dossiers recommandée

```
tp1/
├── database/
│   ├── Dockerfile
│   ├── 01-CreateScheme.sql
│   └── 02-InsertData.sql
├── backend/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
├── httpd/
│   ├── Dockerfile
│   ├── httpd.conf
│   └── index.html
├── docker-compose.yml
└── README.md
```

---

## Checkpoints

1. Database PostgreSQL fonctionnelle avec Adminer
2. Backend API Spring Boot avec endpoint `/departments/IRC/students`
3. Reverse proxy Apache fonctionnel
4. Application 3-tiers complète avec docker-compose
5. Images publiées sur Docker Hub
