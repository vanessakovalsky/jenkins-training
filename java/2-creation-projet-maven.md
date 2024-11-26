# Exercice : Mise en Place d'un Pipeline CI/CD pour Projet Java avec Maven et Jenkins

## Objectifs Pédagogiques
- Créer un projet Java Maven
- Configurer un pipeline Jenkins pour l'intégration continue
- Automatiser les étapes de build, test et déploiement
- Comprendre les pratiques d'intégration continue avec Java

## Prérequis
- Jenkins installé
- Maven installé
- JDK configuré
- Un dépôt Git (GitHub, GitLab, etc.)
- Connaissances de base en Java et Maven

## Structure du Projet Java

### 1. Création du Projet Maven
```bash
mvn archetype:generate -DgroupId=com.example.app \
    -DartifactId=mon-application-java \
    -DarchetypeArtifactId=maven-archetype-quickstart \
    -DinteractiveMode=false
```

### 2. Structure de Projet Recommandée
```
mon-application-java/
├── pom.xml
├── src
│   ├── main
│   │   └── java
│   │       └── com
│   │           └── example
│   │               └── app
│   │                   └── App.java
│   └── test
│       └── java
│           └── com
│               └── example
│                   └── app
│                       └── AppTest.java
└── Jenkinsfile
```

## Configuration du `pom.xml`
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

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.0.0-M5</version>
            </plugin>
        </plugins>
    </build>
</project>
```

## Création du Jenkinsfile
```groovy
pipeline {
    agent any

    tools {
        maven 'Maven 3.8.1'
        jdk 'JDK 11'
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

        stage('Deploy') {
            steps {
                // Exemple de déploiement (à adapter)
                sh 'cp target/*.jar /path/to/deployment/directory/'
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: 'target/*.jar', 
                             fingerprint: true
            
            emailext (
                subject: "Build Réussi: ${currentBuild.fullDisplayName}",
                body: "Le build a été effectué avec succès.",
                to: 'equipe@exemple.com'
            )
        }
        
        failure {
            emailext (
                subject: "Build Échoué: ${currentBuild.fullDisplayName}",
                body: "Le build a échoué. Veuillez vérifier les logs.",
                to: 'equipe@exemple.com'
            )
        }
    }
}
```

## Configuration de Jenkins

### Étapes de Configuration
1. Installer les plugins nécessaires :
   - Maven Integration
   - Git
   - JDK Tools

2. Configurer les outils :
   - "Manage Jenkins" > "Global Tool Configuration"
   - Ajouter Maven et JDK

3. Créer un nouveau Pipeline Jenkins
   - Nouveau Job > Pipeline
   - Configurer le SCM (Git)
   - Sélectionner le Jenkinsfile du projet

## Bonnes Pratiques
- Utiliser des credentials sécurisés
- Gérer les versions avec Git
- Automatiser les tests
- Mettre en place des notifications
- Configurer des environnements distincts (dev, test, prod)

## Exercices Supplémentaires
1. Ajouter des tests d'intégration
2. Implémenter une analyse de code statique (SonarQube)
3. Mettre en place du déploiement conditionnel
4. Intégrer des tests de performance

## Ressources Complémentaires
- Documentation Maven : https://maven.apache.org/guides/
- Documentation Jenkins : https://www.jenkins.io/doc/
- Documentation JUnit : https://junit.org/
```