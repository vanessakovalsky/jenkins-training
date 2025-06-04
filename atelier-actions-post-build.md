# Atelier  : Actions post-build complètes

**Objective :** Configurer un pipeline complet avec actions post-build

## Étape 1 : Script de build avec artefacts

* Créer un nouveau projet à partir du dépôt Nodejs-app utilisé lors d'un précédent exercice.
* Ajouter en étape de build le script suivant : 

```bash
#!/bin/bash
# build-with-artifacts.sh

BUILD_DIR="dist"
REPORTS_DIR="reports"
DOCS_DIR="docs"

echo "🏗️ Build avec génération d'artefacts"

# Nettoyage
rm -rf ${BUILD_DIR} ${REPORTS_DIR} ${DOCS_DIR}
mkdir -p ${BUILD_DIR} ${REPORTS_DIR} ${DOCS_DIR}

# Build de l'application
npm run build
cp -r build/* ${BUILD_DIR}/

# Génération des rapports
echo "📊 Génération des rapports"

# Tests avec couverture
npm run test:coverage
cp -r coverage/* ${REPORTS_DIR}/

# Documentation
npm run docs
cp -r documentation/* ${DOCS_DIR}/

# Analyse de sécurité
npm audit --json > ${REPORTS_DIR}/security-audit.json || true

# Métadonnées du build
cat > ${BUILD_DIR}/build-info.json << EOF
{
  "buildNumber": "${BUILD_NUMBER}",
  "buildUrl": "${BUILD_URL}",
  "gitCommit": "$(git rev-parse HEAD)",
  "gitBranch": "$(git rev-parse --abbrev-ref HEAD)",
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "nodeVersion": "$(node --version)",
  "npmVersion": "$(npm --version)"
}
EOF

echo "✅ Artefacts générés avec succès"
```

## Étape 2 : Configuration Jenkins avancée

* Dans la configuration du projet, ajouter les actions post-builds suivantes :

```yaml
# Post-build Actions dans Jenkins

1. Archive the artifacts:
   Files to archive: "dist/**,reports/**,docs/**"
   
2. Publish HTML reports:
   HTML directory to archive: "reports"
   Index page: "index.html"
   Report title: "Build Reports"
   
3. Publish JUnit test result report:
   Test report XMLs: "reports/junit.xml"
   
4. Record fingerprints:
   Files to fingerprint: "dist/**/*.js,dist/**/*.css"
   
5. Git Publisher:
   Push Only If Build Succeeds: true
   Tags to push: "build-${BUILD_NUMBER}"
```

* Exécuter le pipeline
* Que se passe t'il de plus ?
