# Exercice : Publication d'Artefacts Maven sur un Dépôt Distant via Jenkins

## Objectifs Pédagogiques
- Configurer un pipeline Jenkins pour publier des artefacts Maven
- Gérer les credentials de déploiement
- Comprendre les mécanismes de publication de packages Maven
- Mettre en place un déploiement sécurisé sur un dépôt distant

## Prérequis
- Projet Maven existant
- Jenkins configuré
- Accès à un dépôt Maven (Nexus, Artifactory, Maven Central)
- Connaissances de base en Maven et Jenkins

## Configuration du Projet Maven

### 1. Mise à jour du `pom.xml`
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example.app</groupId>
    <artifactId>mon-application-java</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <distributionManagement>
        <!-- Configuration pour un dépôt Nexus -->
        <repository>
            <id>nexus-releases</id>
            <name>Releases</name>
            <url>https://nexus.exemple.com/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>nexus-snapshots</id>
            <name>Snapshots</name>
            <url>https://nexus.exemple.com/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-deploy-plugin</artifactId>
                <version>3.0.0-M1</version>
            </plugin>
        </plugins>
    </build>
</project>
```

## Configuration de Jenkins

### 2. Configuration des Credentials
1. Dans Jenkins : "Manage Credentials"
2. Ajouter de nouvelles credentials :
   - Type : Username with password
   - ID : `nexus-credentials`
   - Username : votre-utilisateur-nexus
   - Password : votre-mot-de-passe-nexus

### 3. Jenkinsfile Complet
```groovy
pipeline {
    agent any

    tools {
        maven 'Maven 3.8.1'
        jdk 'JDK 11'
    }

    environment {
        // Récupération des credentials Nexus
        NEXUS_CREDENTIALS = credentials('nexus-credentials')
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

        stage('Deploy to Nexus') {
            steps {
                script {
                    // Déploiement conditionnel basé sur le type de version
                    def version = sh(
                        script: 'mvn help:evaluate -Dexpression=project.version -q -DforceStdout',
                        returnStdout: true
                    ).trim()

                    if (version.endsWith('-SNAPSHOT')) {
                        sh '''
                            mvn deploy:deploy-file \
                            -DgroupId=com.example.app \
                            -DartifactId=mon-application-java \
                            -Dversion=${version} \
                            -Dfile=target/mon-application-java-${version}.jar \
                            -Durl=https://nexus.exemple.com/repository/maven-snapshots/ \
                            -DrepositoryId=nexus-snapshots \
                            -Dusername=$NEXUS_CREDENTIALS_USR \
                            -Dpassword=$NEXUS_CREDENTIALS_PSW
                        '''
                    } else {
                        sh '''
                            mvn deploy:deploy-file \
                            -DgroupId=com.example.app \
                            -DartifactId=mon-application-java \
                            -Dversion=${version} \
                            -Dfile=target/mon-application-java-${version}.jar \
                            -Durl=https://nexus.exemple.com/repository/maven-releases/ \
                            -DrepositoryId=nexus-releases \
                            -Dusername=$NEXUS_CREDENTIALS_USR \
                            -Dpassword=$NEXUS_CREDENTIALS_PSW
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: 'target/*.jar', 
                             fingerprint: true
            
            emailext (
                subject: "Déploiement Réussi: ${currentBuild.fullDisplayName}",
                body: "L'artefact a été déployé avec succès sur Nexus.",
                to: 'equipe@exemple.com'
            )
        }
        
        failure {
            emailext (
                subject: "Échec de Déploiement: ${currentBuild.fullDisplayName}",
                body: "Le déploiement de l'artefact a échoué. Veuillez vérifier les logs.",
                to: 'equipe@exemple.com'
            )
        }
    }
}
```

## Configuration de `settings.xml`
Pour une configuration plus robuste, créez un fichier `settings.xml` dans le répertoire `.m2` :

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              https://maven.apache.org/xsd/settings-1.0.0.xsd">
    <servers>
        <server>
            <id>nexus-releases</id>
            <username>${env.NEXUS_CREDENTIALS_USR}</username>
            <password>${env.NEXUS_CREDENTIALS_PSW}</password>
        </server>
        <server>
            <id>nexus-snapshots</id>
            <username>${env.NEXUS_CREDENTIALS_USR}</username>
            <password>${env.NEXUS_CREDENTIALS_PSW}</password>
        </server>
    </servers>
</settings>
```

## Bonnes Pratiques
- Utiliser des credentials sécurisés
- Distinguer les dépôts de snapshots et de releases
- Configurer des politiques de rétention des artefacts
- Sécuriser l'accès au dépôt Maven
- Versionner correctement vos artefacts

## Exercices Supplémentaires
1. Implémenter la gestion des versions semantic versioning
2. Ajouter des vérifications de qualité de code avant le déploiement
3. Mettre en place un cache Maven pour accélérer les builds
4. Configurer des profils de déploiement différents

## Ressources Complémentaires
- Documentation Maven Deploy Plugin : https://maven.apache.org/plugins/maven-deploy-plugin/
- Guide de déploiement Maven : https://maven.apache.org/guides/mini/guide-deployment/
```

Cet exercice détaille les étapes pour déployer des artefacts Maven sur un dépôt distant (comme Nexus ou Artifactory) via un pipeline Jenkins. Il couvre la configuration du `pom.xml`, la gestion des credentials, et un Jenkinsfile complet qui gère différemment les versions snapshot et release.

Voulez-vous que je vous explique plus en détail un aspect spécifique de cet exercice ?