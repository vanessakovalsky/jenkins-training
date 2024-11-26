# Exercice : Configuration d'un Cluster Jenkins avec Docker

**Objectif :** 
Mettre en place un cluster Jenkins avec un nœud maître (master) et plusieurs nœuds agents (workers) en utilisant Docker Compose.

**Prérequis :**
- Docker installé
- Docker Compose installé
- Connaissance de base de Jenkins et Docker

**Étapes détaillées :**

Je vais détailler l'exercice de configuration du cluster Jenkins avec Docker de manière comprehensive.

1. **Préparation de l'environnement**

Structure du projet :
```
jenkins-cluster/
│
├── docker-compose.yml
├── jenkins-master/
│   ├── Dockerfile
│   └── jenkins-config/
│
├── jenkins-agent/
│   ├── Dockerfile
│   └── start-agent.sh
│
└── docker-network.yml
```

2. **Docker Compose Configuration**



```yaml
version: '3.8'

networks:
  jenkins-network:
    driver: bridge

volumes:
  jenkins-master-data:
  jenkins-agent-data:

services:
  jenkins-master:
    image: jenkins/jenkins:lts
    container_name: jenkins-master
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins-master-data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JENKINS_OPTS="--prefix=/jenkins"
    networks:
      - jenkins-network
    restart: always

  jenkins-agent-1:
    build: 
      context: ./jenkins-agent
      dockerfile: Dockerfile
    container_name: jenkins-agent-1
    volumes:
      - jenkins-agent-data:/home/jenkins
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - jenkins-network
    environment:
      - JENKINS_MASTER_URL=http://jenkins-master:8080/jenkins
    depends_on:
      - jenkins-master
    restart: always

  jenkins-agent-2:
    build: 
      context: ./jenkins-agent
      dockerfile: Dockerfile
    container_name: jenkins-agent-2
    volumes:
      - jenkins-agent-data:/home/jenkins
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - jenkins-network
    environment:
      - JENKINS_MASTER_URL=http://jenkins-master:8080/jenkins
    depends_on:
      - jenkins-master
    restart: always

```

3. **Dockerfile pour l'agent Jenkins**

```dockerfile
FROM ubuntu:20.04

# Éviter les interactions pendant l'installation
ENV DEBIAN_FRONTEND=noninteractive

# Installer les dépendances
RUN apt-get update && apt-get install -y \
    openjdk-11-jdk \
    openssh-server \
    git \
    curl \
    wget \
    docker.io \
    && rm -rf /var/lib/apt/lists/*

# Configuration SSH
RUN mkdir /var/run/sshd
RUN useradd -m -s /bin/bash jenkins
RUN echo 'jenkins:jenkins' | chpasswd
RUN usermod -aG docker jenkins

# Autoriser la connexion SSH
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config

# Script de démarrage de l'agent
COPY start-agent.sh /start-agent.sh
RUN chmod +x /start-agent.sh

EXPOSE 22

CMD ["/start-agent.sh"]

```

4. **Script de démarrage de l'agent**

```bash
#!/bin/bash

# Démarrer le service SSH
service ssh start

# Configuration automatique de l'agent Jenkins
if [ -n "$JENKINS_MASTER_URL" ]; then
    echo "Connexion au serveur Jenkins Master: $JENKINS_MASTER_URL"
    # Ici, vous pouvez ajouter la logique de connexion automatique de l'agent
    # Par exemple, utiliser jenkins-agent.jar ou un script de découverte
fi

# Maintenir le conteneur actif
tail -f /dev/null

```

**Instructions de configuration et de déploiement :**

1. Préparez les fichiers dans la structure de projet mentionnée
2. Configurez les permissions : 
   ```bash
   chmod +x jenkins-agent/start-agent.sh
   ```

3. Déployez le cluster :
   ```bash
   docker-compose up -d --build
   ```


**Points importants :**
- Le cluster utilise un réseau Docker personnalisé
- Volumes persistants pour les données Jenkins
- Configuration SSH pour la communication inter-nœuds
- Agents configurés dynamiquement

**Étapes supplémentaires recommandées :**
- Configurer l'authentification SSH entre le maître et les agents
- Mettre en place des credentials Jenkins sécurisés
- Utiliser des secrets Docker pour la gestion des mots de passe

**Améliorations possibles :**
- Ajouter un proxy Nginx
- Implémenter des stratégies de sauvegarde
- Mettre en place une surveillance (Prometheus/Grafana)

## Utilisation du cluster pour répartir l'exécution des jobs

Je vais détailler un exemple complet de configuration des Labels et Affinités dans Jenkins, avec des cas d'usage pratiques.

**Exemple Détaillé : Configuration des Labels et Affinités**

1. **Scénario d'Infrastructure**

Imaginons une entreprise de développement logiciel avec différents types de projets :
- Projets web (frontend et backend)
- Projets mobile 
- Projets de data science
- Projets de tests et d'intégration continue

2. **Configuration des Agents avec Labels**

```groovy
// Configuration des labels pour différents types d'agents

// Agent pour développement web
node('web && frontend') {
    stage('Frontend Build') {
        // Étapes de build spécifiques au frontend
        tools {
            nodejs 'NodeJS 14'
        }
    }
}

// Agent pour développement backend
node('web && backend && java') {
    stage('Backend Build') {
        // Étapes de build spécifiques au backend
        tools {
            jdk 'JDK 11'
            maven 'Maven 3'
        }
    }
}

// Agent pour projets mobile
node('mobile && android') {
    stage('Mobile App Build') {
        // Étapes de build pour application mobile
        tools {
            androidSdk 'Android SDK'
        }
    }
}

// Agent pour data science
node('data && python') {
    stage('Data Science Pipeline') {
        // Traitement de données et machine learning
        tools {
            python 'Python 3.8'
            conda 'Anaconda'
        }
    }
}

```

3. **Exemple de Pipeline Complet**

```groovy
pipeline {
    agent none  // Configuration d'agent dynamique

    stages {
        stage('Validation Code') {
            // Vérification du code sur n'importe quel agent Linux
            agent { 
                label 'linux && code-quality' 
            }
            steps {
                sh 'run-code-linters.sh'
                sh 'run-security-checks.sh'
            }
        }

        stage('Build Frontend') {
            // Build spécifique aux projets frontend
            agent { 
                label 'web && frontend && nodejs' 
            }
            steps {
                nodejs(nodeJSInstallationName: 'NodeJS 14') {
                    sh 'npm install'
                    sh 'npm run build'
                }
            }
        }

        stage('Build Backend') {
            // Build spécifique aux projets backend
            agent { 
                label 'web && backend && java' 
            }
            steps {
                withMaven(maven: 'Maven 3') {
                    sh 'mvn clean package'
                }
            }
        }

        stage('Tests Intégration') {
            // Tests sur un agent avec des capacités de test
            agent { 
                label 'integration-test && docker' 
            }
            steps {
                sh 'docker-compose up -d test-environment'
                sh 'run-integration-tests.sh'
            }
        }

        stage('Déploiement') {
            // Déploiement sur un agent avec droits de déploiement
            agent { 
                label 'deployment && production' 
            }
            steps {
                sh 'deploy-to-production.sh'
            }
        }
    }
}

```

4. **Configuration Manuelle des Labels**

Dans l'interface Jenkins :
- Aller dans "Manage Jenkins" > "Manage Nodes and Clouds"
- Sélectionner un agent
- Cliquer sur "Configure"
- Section "Labels" : ajouter des labels séparés par des espaces

5. **Script de Configuration Programmatique**

```groovy
import jenkins.model.*
import hudson.model.*

// Fonction pour configurer les labels d'un agent
def configureNodeLabels(String nodeName, List<String> labels) {
    def jenkins = Jenkins.getInstance()
    def node = jenkins.getNode(nodeName)
    
    if (node != null) {
        node.setLabelString(labels.join(' '))
        node.save()
        println "Labels mis à jour pour $nodeName: ${labels.join(', ')}"
    } else {
        println "Nœud $nodeName non trouvé"
    }
}

// Exemple d'utilisation
configureNodeLabels('agent-01', [
    'linux', 
    'web', 
    'frontend', 
    'nodejs'
])

configureNodeLabels('agent-02', [
    'linux', 
    'backend', 
    'java', 
    'maven'
])

```

**Bonnes Pratiques pour les Labels**

1. **Conventions de Nommage**
- Utilisez des labels courts et explicites
- Privilégiez les minuscules
- Utilisez des traits d'union si nécessaire

2. **Stratégies de Labellisation**
- Systèmes d'exploitation : `linux`, `windows`
- Langages : `java`, `python`, `nodejs`
- Types de projets : `web`, `mobile`, `backend`
- Environnements : `dev`, `test`, `production`
- Capacités : `docker`, `kubernetes`

3. **Gestion des Contraintes**
- Un job peut requérir plusieurs labels
- Les labels sont ET logiques (`&&`)
- Possibilité d'exclure des labels (`!`)

**Exemple d'Exclusion**

```groovy
// Sélectionne un agent Linux qui n'est PAS un agent de production
node('linux && !production') {
    // Étapes de build
}
```

**Points Clés**
- Les labels permettent un routage intelligent des jobs
- Ils offrent une granularité fine dans l'allocation des ressources
- Facilitent l'utilisation optimale de l'infrastructure