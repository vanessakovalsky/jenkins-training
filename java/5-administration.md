# Exercice d'Administration Jenkins

**Objectif :** 
L'objectif de l'exercice est de permettre aux participants de maîtriser l'administration complète de Jenkins en configurant une infrastructure CI/CD sécurisée, performante et hautement automatisée, couvrant l'installation, la gestion des agents, la sécurité et les pipelines de déploiement.

**Prérequis :**
- Docker installé
- Docker Compose installé
- Connaissance de base de Jenkins et Docker

## Étape 1 : Configuration Initiale et Sécurité

### 1.1 Configuration de la Sécurité
1. Configurer l'authentification :
   - Activer la sécurité globale
   - Choisir un mode d'authentification :
     * Jenkins User Database
     * LDAP
     * Active Directory
     * OAuth/SSO

2. Créer des stratégies de permissions :
   - Définir des rôles :
     * Administrateurs
     * Développeurs
     * Auditeurs
   - Configurer des matrices de permissions granulaires


### Tâches Pratiques
- Créer 3 utilisateurs avec des rôles différents
- Configurer des permissions spécifiques
- Mettre en place une authentification sécurisée


### Partie 2 : Gestion des Agents et du Cloud


## Étape 2 : Gestion des Agents et du Cloud

### 2.1 Configuration des Agents
1. Créer des agents avec différents labels
   - Agent Linux pour builds Java
   - Agent Windows pour tests .NET
   - Agent Docker pour conteneurisation

2. Configuration des méthodes de connexion
   - SSH
   - JNLP
   - Docker


### 2.2 Configuration Cloud
1. Intégration Docker
2. Configuration Kubernetes
3. Configuration AWS/Azure Cloud Agents

### Tâches Pratiques
- Configurer 3 types différents d'agents
- Implémenter une stratégie de répartition de charge
- Mettre en place un agent Cloud dynamique


### Partie 3 : Gestion des Plugins et Maintenance


## Étape 3 : Plugins et Maintenance

### 3.1 Gestion des Plugins
1. Installation de plugins essentiels
   - Blue Ocean
   - Pipeline
   - Docker
   - Git
   - Credentials

2. Mise à jour et maintenance

### Script de Gestion des Plugins

```groovy
import jenkins.model.*
import hudson.model.*
import hudson.plugins.git.*

def jenkins = Jenkins.getInstance()
def pluginManager = jenkins.getPluginManager()
def updateCenter = jenkins.getUpdateCenter()

// Liste des plugins à installer
def requiredPlugins = [
    'git', 
    'docker-plugin', 
    'pipeline-job', 
    'credentials'
]

requiredPlugins.each { pluginName ->
    if (!pluginManager.getPlugin(pluginName)) {
        updateCenter.getPlugin(pluginName).install()
    }
}

// Sauvegarde de la configuration
jenkins.save()
```

### 3.2 Stratégies de Sauvegarde
1. Configuration de la sauvegarde périodique
2. Sauvegarde des configurations
3. Restauration

### Script de Sauvegarde

```bash
#!/bin/bash
JENKINS_HOME="/var/jenkins_home"
BACKUP_DIR="/backup/jenkins"
DATE=$(date +"%Y%m%d_%H%M%S")

# Créer un backup complet
tar -czvf "${BACKUP_DIR}/jenkins-backup-${DATE}.tar.gz" ${JENKINS_HOME}

# Rotation des backups (garder les 7 derniers)
cd ${BACKUP_DIR}
ls -t jenkins-backup-*.tar.gz | tail -n +8 | xargs rm -f
```

### Tâches Pratiques
- Installer et configurer 5 plugins
- Mettre en place un script de sauvegarde automatique
- Tester la restauration à partir d'une sauvegarde

### Partie 4 : Conception de Pipelines CI/CD


// Pipeline CI/CD Complet
pipeline {
    agent {
        label 'linux && docker'
    }
    
    environment {
        DOCKER_REGISTRY = 'mon-registry.com'
        APP_NAME = 'mon-application'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'develop', 
                    credentialsId: 'git-credentials', 
                    url: 'https://github.com/monorg/monapp.git'
            }
        }
        
        stage('Build') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}")
                }
            }
        }
        
        stage('Tests') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm test'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'npm run integration-test'
                    }
                }
            }
        }
        
        stage('Déploiement') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry('https://${DOCKER_REGISTRY}', 'docker-credentials') {
                        docker.image("${APP_NAME}:${BUILD_NUMBER}").push()
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Déploiement réussi !'
            // Notifications, etc.
        }
        failure {
            echo 'Échec du déploiement'
            // Gestion des erreurs
        }
    }
}
```


**Points Clés de l'Exercice**
1. Couvre tous les aspects essentiels de l'administration Jenkins
2. Approche pratique et progressive
3. Met l'accent sur la sécurité et l'automatisation
4. Permet une compréhension globale de Jenkins
