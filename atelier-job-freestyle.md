# Création d'un job freestyle


**Objectif :** 

- Préparer un environnement Jenkins complet avec Docker
- Créer un job de build pour une application Node.js

#### Étape 1 : Configuration Docker Compose

Créez le fichier `docker-compose.yml` :

```yaml
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins-master
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
    networks:
      - jenkins-network

  jenkins-agent:
    image: jenkins/ssh-agent:latest
    container_name: jenkins-agent
    environment:
      - JENKINS_AGENT_SSH_PUBKEY=ssh-rsa AAAAB3NzaC1yc2E...
    networks:
      - jenkins-network

  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    ports:
      - "9000:9000"
    environment:
      - SONAR_JDBC_URL=jdbc:postgresql://db:5432/sonar
      - SONAR_JDBC_USERNAME=sonar
      - SONAR_JDBC_PASSWORD=sonar
    networks:
      - jenkins-network

  db:
    image: postgres:13
    container_name: postgres
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
      - POSTGRES_DB=sonar
    volumes:
      - postgresql_data:/var/lib/postgresql/data
    networks:
      - jenkins-network

volumes:
  jenkins_home:
  postgresql_data:

networks:
  jenkins-network:
    driver: bridge
```

#### Étape 2 : Démarrage de l'environnement

```bash
# Démarrer l'environnement
docker-compose up -d

# Vérifier les conteneurs
docker-compose ps

# Récupérer le mot de passe admin
docker exec jenkins-master cat /var/jenkins_home/secrets/initialAdminPassword
```

#### Étape 3 : Préparation du projet de test

Créez un projet Node.js simple :

* Créer un dossier et se mettre à l'intérieur
* Les deux commandes suivantes vont créer les fichiers de l'application que l'on va builder.

```bash
# Créer la structure du projet
mkdir nodejs-app && cd nodejs-app

# Fichier package.json
cat > package.json << EOF
{
  "name": "jenkins-demo-app",
  "version": "1.0.0",
  "description": "Application de démonstration Jenkins",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "mocha test/",
    "lint": "eslint .",
    "build": "echo 'Build completed'"
  },
  "dependencies": {
    "express": "^4.18.0"
  },
  "devDependencies": {
    "mocha": "^10.0.0",
    "chai": "^4.3.0",
    "eslint": "^8.0.0"
  }
}
EOF

# Fichier app.js
cat > app.js << EOF
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({ message: 'Hello Jenkins!', version: '1.0.0' });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'OK', timestamp: new Date().toISOString() });
});

if (require.main === module) {
  app.listen(port, () => {
    console.log(\`App listening at http://localhost:\${port}\`);
  });
}

module.exports = app;
EOF

# Tests unitaires
mkdir test
cat > test/app.test.js << EOF
const chai = require('chai');
const request = require('supertest');
const app = require('../app');

const expect = chai.expect;

describe('Application Tests', () => {
  it('should return hello message', (done) => {
    request(app)
      .get('/')
      .expect(200)
      .end((err, res) => {
        if (err) return done(err);
        expect(res.body.message).to.equal('Hello Jenkins!');
        done();
      });
  });

  it('should return health status', (done) => {
    request(app)
      .get('/health')
      .expect(200)
      .end((err, res) => {
        if (err) return done(err);
        expect(res.body.status).to.equal('OK');
        done();
      });
  });
});
EOF
```

* Créer on se connecter sur github (https://github.com/)
* Créer un projet public appelé nodejs-app
* Pousser les fichiers créé sur le dépôt

#### Étape 4 : Configuration du job Jenkins

1. **Accédez à Jenkins** : http://localhost:8080
2. **Créer un nouveau job** :
   - Cliquez sur "Nouvel Item"
   - Nom : `nodejs-app-build`
   - Type : "Projet free-style"

3. **Configuration du job** :

```yaml
# Configuration générale
Description: "Build et test de l'application Node.js de démonstration"

# Gestion du code source
Source Code Management: Git
Repository URL: https://github.com/votre-username/nodejs-app.git
Branch Specifier: */main

# Déclencheurs de build
Build Triggers:
  - Poll SCM: H/5 * * * *  # Vérifier toutes les 5 minutes
  - GitHub hook trigger for GITScm polling

# Environnement de build
Build Environment:
  - Delete workspace before build starts
  - Add timestamps to the Console Output
```

* Enregistrer le job

#### Étape 5 : Déclencheur Poll SCM

```cron
# Syntaxe cron Jenkins
# minute hour day month dayOfWeek

# Exemples pratiques
H/15 * * * *     # Toutes les 15 minutes
H 2 * * *        # Tous les jours à 2h du matin  
H H(0-7) * * *   # Une fois par jour entre 0h et 7h
H H * * 0        # Une fois par semaine le dimanche
```

#### Étape 6 : Configuration webhook

1. **Dans GitHub** :
   - Settings → Webhooks → Add webhook
   - Payload URL : `http://your-jenkins:8080/github-webhook/`
   - Content type : `application/json`
   - Events : `Just the push event`

2. **Dans Jenkins** :
   - Cocher "GitHub hook trigger for GITScm polling"
  

#### Étape 6 : Dockerfile pour l'application

* Créer dans votre projet un fichier Dockerfile avec le contenu suivant :

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

# Copie des fichiers de dépendances
COPY package*.json ./

# Installation des dépendances
RUN npm ci --only=production

# Copie du code source
COPY . .

# Exposition du port
EXPOSE 3000

# Utilisateur non-root pour la sécurité
USER node

# Commande de démarrage
CMD ["npm", "start"]
```

* Pensez à commit et push le fichier

#### Étape 7 : Configuration du build Jenkins

* Ajouter une étape dans votre job qui exécute les commandes shell suivantes :

```bash
#!/bin/bash
set -e  # Arrêt en cas d'erreur

echo "🚀 Début du build - Build #${BUILD_NUMBER}"

# Variables d'environnement
APP_NAME="nodejs-app"
IMAGE_NAME="${APP_NAME}"
IMAGE_TAG="${BUILD_NUMBER}"
LATEST_TAG="latest"

echo "📦 Nettoyage de l'espace de travail"
docker system prune -f || true

echo "📋 Vérification de l'environnement"
echo "Node version: $(node --version)"
echo "NPM version: $(npm --version)"
echo "Docker version: $(docker --version)"

echo "📥 Installation des dépendances"
npm ci

echo "🔍 Analyse statique du code"
npm run lint

echo "🧪 Exécution des tests unitaires"
npm test -- --reporter=xunit --output-file=test-results.xml

echo "📊 Génération du rapport de couverture"
npm run test:coverage || true

echo "🏗️ Build de l'application"
npm run build

echo "🐳 Construction de l'image Docker"
docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:${LATEST_TAG}

echo "🔒 Scan de sécurité de l'image"
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} || true

echo "✅ Build terminé avec succès"
```

#### Étape 7 : Lancement du build

1. Accédez au job `nodejs-app-build`
2. Cliquez sur "Lancer un build"
3. Observez la console output en temps réel

#### Étape 2 : Analyse des résultats

```bash
# Vérification des artefacts générés
ls -la workspace/
ls -la builds/${BUILD_NUMBER}/

# Consultation des logs
cat builds/${BUILD_NUMBER}/log

# Vérification de l'image Docker
docker images | grep nodejs-app
```
