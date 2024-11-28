# Pipeline CI/CD avec Déploiement Docker Local

## Objectifs Pédagogiques
- Intégrer la conteneurisation Docker dans un pipeline Jenkins
- Construire et publier une image Docker
- Automatiser le déploiement d'une application Java
- Mettre en place un pipeline d'intégration continue complet

## Prérequis
- Projet Java Maven existant
- Jenkins installé
- Docker installé sur le serveur Jenkins
- Accès à un registre Docker (Docker Hub, Nexus, etc.)

## Structure du Projet
```
mon-application-java/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
├── Dockerfile
└── Jenkinsfile
```

## Création du Dockerfile
```dockerfile
# Étape de construction
FROM maven:3.8.1-openjdk-11-slim AS build
WORKDIR /app

# Copier les fichiers de configuration Maven
COPY pom.xml .
COPY src ./src

# Construire l'application
RUN mvn clean package -DskipTests=true

# Étape finale
FROM openjdk:11-jre-slim
WORKDIR /app

# Créer un répertoire pour les données persistantes
RUN mkdir -p /app/data

# Copier le jar construit
COPY --from=build /app/target/*.jar app.jar

# Configuration du point d'entrée
ENTRYPOINT ["java", "-jar", "/app/app.jar"]

# Documentation du port exposé
EXPOSE 8080

# Définir un volume pour la persistance
VOLUME ["/app/data"]
```

## Ajout dans le Jenkinsfile
```groovy
pipeline {
    agent any

    tools {
        maven 'Maven 3.8.1'
    }

    environment {
        DOCKER_IMAGE = "mon-organisation/mon-application-java"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        // Nom du réseau Docker
        DOCKER_NETWORK = "jenkins-network"
        // Nom du conteneur de l'application
        APP_CONTAINER_NAME = "mon-application-java"
    }

    stages {
        stage('Préparation du Réseau Docker') {
            steps {
                script {
                    // Créer un réseau Docker s'il n'existe pas
                    sh '''
                        docker network inspect ${DOCKER_NETWORK} || \
                        docker network create ${DOCKER_NETWORK}
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Construction de l'image Docker
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                }
            }
        }

        stage('Arrêt du Conteneur Existant') {
            steps {
                script {
                    // Arrêter et supprimer le conteneur existant
                    sh '''
                        docker stop ${APP_CONTAINER_NAME} || true
                        docker rm ${APP_CONTAINER_NAME} || true
                    '''
                }
            }
        }

        stage('Déploiement Local') {
            steps {
                script {

                    // Démarrage du nouveau conteneur
                    sh """
                        docker run -d \
                            --name ${APP_CONTAINER_NAME} \
                            --network ${DOCKER_NETWORK} \
                            -p 8080:8080 \
                            --restart unless-stopped \
                            ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }

        stage('Nettoyage') {
            steps {
                script {
                    // Supprimer les anciennes images
                    sh '''
                        docker image prune -f
                        docker images | grep ${DOCKER_IMAGE} | grep -v ${DOCKER_TAG} | awk \'{print $3}\' | xargs -r docker rmi || true
                    '''
                }
            }
        }
    }
```

## Configuration Requise

### 0.Configuration de votre environnement

- Afin de préparer l'environnement d'exécution pour pouvoir exécuter les commandes dockers, nous allons devoir faire un peu de préparation
- Commencer par arrêter votre conteneur jenkins : `docker stop jenkins`
- Ensuite récupérer les fichiers 
    - Dockerfile à cette adresse : https://github.com/vanessakovalsky/jenkins-training/blob/main/docker/Dockerfile
    - docker-compose.yml à cette adresse : https://github.com/vanessakovalsky/jenkins-training/blob/main/docker/docker-compose.yml 
- Ouvrir un terminal (ou un powershell) dans le dossier où se trouve les deux fichiers
- Exécuter la commande : `docker compose up -d`
- Une fois le conteneur construit et lancé, vous devriez pouvoir accéder à votre jenkins sur : http://localhost:8080 (votre identifiant/mot de passe est le même que celui que vous aviez défini, nous avons réutiliser le volume crée dans l'exercice 1)

### 1. Credentials Jenkins
- `docker-registry-credentials` : Identifiants Docker Hub ou autre registre

### 2. Plugins Jenkins Nécessaires
- Docker Pipeline
- Credentials
- Email Extension

## Bonnes Pratiques
- Utiliser des tags de version sémantique
- Isoler les environnements (staging, production)
- Configurer des volumes pour la persistance des données
- Mettre en place des tests d'intégration
- Gérer les secrets de manière sécurisée

## Exercices Supplémentaires
1. Ajouter des tests de sécurité à l'image Docker
2. Implémenter la gestion des configurations avec Docker Compose
3. Mettre en place un mécanisme de rollback
4. Ajouter des checks de santé avancés

## Ressources Complémentaires
- Documentation Docker : https://docs.docker.com/
- Jenkins Pipeline : https://www.jenkins.io/doc/book/pipeline/


