# Atelier  : Intégration complète avec SonarQube

**Objective :** Intégrer SonarQube pour l'analyse de qualité du code

## Étape 1 : Configuration SonarQube

* Vous avez deux manières de configurer SonarQube : soit en passant par l'interface, soit via des appels à l'API
* Pour l'interface, rendez vous sur http://localhost:9000
* Connectez vous avec les identifiants admin/admin puis définissez un mot de passe
* Dans votre compte (en haut a droit en cliquant sur la lettre du nom de l'utilisateur), générer un token global et noter le. 


* On peut également obtenir le token en bash
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

## Étape 2 : Configuration du projet SonarQube dans Jenkins

* Définir un nouveau projet freestyle avec la même config git que celle du projet nodejs
* Définir une variable d'environement nommée SONAR_AUTH_TOKEN qui contient le token sous le format Secret-texte.
* Ajouter une étape de build avec le script shell suivant :
  
```bash
#!/bin/bash
# sonar-analysis.sh

echo "🔍 Analyse SonarQube en cours..."

# Variables d'environnement
export SONAR_HOST_URL="http://sonarqube:9000"
export SONAR_TOKEN="${SONAR_AUTH_TOKEN}"

# Installation des dépendances

npm install

# Exécution des tests unitaires
npm run test

# Analyse SonarQube
npx sonar-scanner \
  -Dsonar.projectKey=nodejs-app \
  -Dsonar.projectName="NodeJS Application" \
  -Dsonar.projectVersion=${BUILD_NUMBER} \
  -Dsonar.sources=. \
  -Dsonar.tests=test \
  -Dsonar.exclusions=**./*.test.js \
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

* Exécuter le build
* Une fois le build terminé en succès, aller voir dans SonarQube que vos informations qualités et de tests sont bien remontées.
