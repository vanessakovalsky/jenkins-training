# Atelier  : Intégration complète avec SonarQube

**Objective :** Intégrer SonarQube pour l'analyse de qualité du code

## Étape 1 : Configuration SonarQube

```bash
# Démarrage de SonarQube (déjà dans docker-compose.yml)
docker-compose up -d sonarqube

# Attendre que SonarQube soit prêt
echo "⏳ Attente du démarrage de SonarQube..."
until curl -sSf http://localhost:9000/api/system/status | grep -q '"status":"UP"'; do
    sleep 5
done

# Configuration initiale via API
curl -u admin:admin -X POST \
  "http://localhost:9000/api/users/change_password" \
  -d "login=admin&password=admin&previousPassword=admin"

# Création du token d'accès
SONAR_TOKEN=$(curl -u admin:newpassword -X POST \
  "http://localhost:9000/api/user_tokens/generate" \
  -d "name=jenkins" | jq -r '.token')

echo "Token SonarQube: ${SONAR_TOKEN}"
```

## Étape 2 : Configuration du projet SonarQube

```javascript
// sonar-project.js - Configuration pour Node.js
module.exports = {
  serverUrl: 'http://localhost:9000',
  token: process.env.SONAR_TOKEN,
  options: {
    'sonar.projectKey': 'nodejs-app',
    'sonar.projectName': 'NodeJS Application Demo',
    'sonar.projectVersion': process.env.BUILD_NUMBER || '1.0.0',
    'sonar.sources': 'src',
    'sonar.tests': 'test',
    'sonar.javascript.lcov.reportPaths': 'coverage/lcov.info',
    'sonar.testExecutionReportPaths': 'test-results.xml',
    'sonar.coverage.exclusions': [
      'test/**/*',
      'coverage/**/*',
      'node_modules/**/*'
    ].join(','),
    'sonar.cpd.exclusions': [
      'test/**/*'
    ].join(',')
  }
};
```

## Étape 3 : Script d'analyse

* Définir un nouveau projet freestyle avec la même config git sur le projet nodejs

* Ajouter un paramètre d'environnement pour définir le token

```bash
#!/bin/bash
# sonar-analysis.sh

echo "🔍 Analyse SonarQube en cours..."

# Variables d'environnement
export SONAR_HOST_URL="http://sonarqube:9000"
export SONAR_TOKEN="${SONAR_AUTH_TOKEN}"

# Installation des dépendances

npm install

# Exécution des tests avec couverture
npm run test:coverage

# Analyse SonarQube
npx sonar-scanner \
  -Dsonar.projectKey=nodejs-app \
  -Dsonar.projectName="NodeJS Application" \
  -Dsonar.projectVersion=${BUILD_NUMBER} \
  -Dsonar.sources=. \
  -Dsonar.tests=test \
  -Dsonar.exclusions=**./*.test.js \
  -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
  -Dsonar.testExecutionReportPaths=test-results.xml \
  -Dsonar.host.url=http://sonarqube:9000 \
  -Dsonar.token=${SONAR_TOKEN}

# Attente du Quality Gate
echo "⏳ Vérification du Quality Gate..."
sleep 30

# Récupération du statut Quality Gate
QG_STATUS=$(curl -s -u ${SONAR_TOKEN}: \
  "${SONAR_HOST_URL}/api/qualitygates/project_status?projectKey=nodejs-app" \
  | jq -r '.projectStatus.status')

echo "Quality Gate Status: ${QG_STATUS}"

if [ "${QG_STATUS}" != "OK" ]; then
    echo "❌ Quality Gate failed!"
    exit 1
else
    echo "✅ Quality Gate passed!"
fi
```
