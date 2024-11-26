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

# Copier le jar construit
COPY --from=build /app/target/*.jar app.jar

# Configuration du point d'entrée
ENTRYPOINT ["java", "-jar", "/app/app.jar"]

# Documentation du port exposé
EXPOSE 8080
```

## Mise à jour du `pom.xml`
```xml
<project>
    <!-- Configurations précédentes -->
    <properties>
        <docker.image.prefix>votre-organisation</docker.image.prefix>
        <docker.image.name>mon-application-java</docker.image.name>
    </properties>

    <build>
        <plugins>
            <!-- Plugins précédents -->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>dockerfile-maven-plugin</artifactId>
                <version>1.4.13</version>
                <configuration>
                    <repository>${docker.image.prefix}/${docker.image.name}</repository>
                    <tag>${project.version}</tag>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

## Jenkinsfile Complet
```groovy
pipeline {
    agent any

    tools {
        maven 'Maven 3.8.1'
        jdk 'JDK 11'
    }

    environment {
        // Credentials Docker Registry
        DOCKER_REGISTRY_CREDENTIALS = credentials('docker-registry-credentials')
        // Nom et tag de l'image
        DOCKER_IMAGE = "votre-organisation/mon-application-java"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/votre-utilisateur/mon-application-java.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                }
            }
        }

        stage('Push to Docker Registry') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-registry-credentials') {
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push('latest')
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                script {
                    // Arrêter et supprimer le conteneur existant
                    sh """
                        docker stop mon-application-staging || true
                        docker rm mon-application-staging || true
                    """

                    // Démarrer un nouveau conteneur
                    sh """
                        docker run -d \
                            --name mon-application-staging \
                            -p 8080:8080 \
                            ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }

        stage('Integration Tests') {
            steps {
                // Ajouter des tests d'intégration
                sh 'curl http://localhost:8080/health || exit 1'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Déploiement sur le serveur de production
                    sshagent(['production-server-credentials']) {
                        sh """
                            ssh user@production-server \
                            "docker pull ${DOCKER_IMAGE}:${DOCKER_TAG} && \
                             docker stop mon-application-prod || true && \
                             docker rm mon-application-prod || true && \
                             docker run -d \
                                --name mon-application-prod \
                                -p 80:8080 \
                                ${DOCKER_IMAGE}:${DOCKER_TAG}"
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            // Notifications et artefacts
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            
            emailext (
                subject: "Déploiement Réussi: ${currentBuild.fullDisplayName}",
                body: "L'application a été construite, testée et déployée avec succès.",
                to: 'equipe@exemple.com'
            )
        }
        
        failure {
            emailext (
                subject: "Échec de Déploiement: ${currentBuild.fullDisplayName}",
                body: "Le pipeline de déploiement a échoué. Veuillez vérifier les logs.",
                to: 'equipe@exemple.com'
            )
        }

        always {
            // Nettoyer les images Docker locales
            sh 'docker system prune -f'
        }
    }
}
```

## Configuration Requise

### 1. Credentials Jenkins
- `docker-registry-credentials` : Identifiants Docker Hub ou autre registre
- `production-server-credentials` : Clé SSH pour le serveur de production

### 2. Plugins Jenkins Nécessaires
- Docker Pipeline
- SSH Agent
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


## Jenkinsfile 
```groovy
pipeline {
    agent any

    tools {
        maven 'Maven 3.8.1'
        jdk 'JDK 11'
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

        stage('Checkout') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/votre-utilisateur/mon-application-java.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests=true'
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
                    // Création d'un volume pour la persistance des données
                    sh '''
                        docker volume create ${APP_CONTAINER_NAME}-data || true
                    '''

                    // Démarrage du nouveau conteneur
                    sh """
                        docker run -d \
                            --name ${APP_CONTAINER_NAME} \
                            --network ${DOCKER_NETWORK} \
                            -p 8080:8080 \
                            -v ${APP_CONTAINER_NAME}-data:/app/data \
                            --restart unless-stopped \
                            ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }

        stage('Vérification du Déploiement') {
            steps {
                script {
                    // Attendre le démarrage du conteneur
                    sh 'sleep 30'
                    
                    // Test de santé basique
                    sh 'curl http://localhost:8080/health || exit 1'
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

    post {
        success {
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            
            emailext (
                subject: "Déploiement Réussi: ${currentBuild.fullDisplayName}",
                body: "L'application a été construite, testée et déployée avec succès.",
                to: 'equipe@exemple.com'
            )
        }
        
        failure {
            emailext (
                subject: "Échec de Déploiement: ${currentBuild.fullDisplayName}",
                body: "Le pipeline de déploiement a échoué. Veuillez vérifier les logs.",
                to: 'equipe@exemple.com'
            )
        }
    }
}
```

## Dockerfile 
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

## Points Clés de la Modification
1. Déploiement sur le même hôte Docker que Jenkins
2. Création d'un réseau Docker pour l'isolation
3. Utilisation d'un volume persistant
4. Gestion du cycle de vie du conteneur
5. Nettoyage des anciennes images

## Bonnes Pratiques
- Utiliser un réseau Docker dédié
- Créer des volumes pour la persistance des données
- Nettoyer régulièrement les images et conteneurs
- Configurer un redémarrage automatique
- Ajouter des tests de santé

## Considérations Supplémentaires
- Adapter le port si nécessaire
- Configurer des variables d'environnement
- Ajouter des mécanismes de rollback si besoin
```

