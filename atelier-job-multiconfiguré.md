# 🔧 Atelier  : Job multi-configuré

**Objective :** Créer un job de test sur plusieurs environnements

## Étape 1 : Préparation des images Docker

```bash
# Créer des images pour différents environnements
cat > Dockerfile.node16 << EOF
FROM node:16-alpine
RUN apk add --no-cache git
WORKDIR /app
CMD ["sh"]
EOF

cat > Dockerfile.node18 << EOF
FROM node:18-alpine
RUN apk add --no-cache git
WORKDIR /app
CMD ["sh"]
EOF

# Build des images
docker build -f Dockerfile.node16 -t test-node:16 .
docker build -f Dockerfile.node18 -t test-node:18 .
```

## Étape 2 : Script de test matriciel

```bash
#!/bin/bash
# test-matrix.sh

NODE_VERSION=${NODE_VERSION:-18}
OS=${OS:-ubuntu-22.04}

echo "🧪 Tests sur Node.js ${NODE_VERSION} - ${OS}"

# Sélection de l'image Docker
case ${NODE_VERSION} in
    16) IMAGE="test-node:16" ;;
    18) IMAGE="test-node:18" ;;
    20) IMAGE="test-node:20" ;;
    *) echo "Version Node.js non supportée"; exit 1 ;;
esac

# Exécution des tests dans le conteneur
docker run --rm -v ${WORKSPACE}:/app ${IMAGE} sh -c "
    cd /app
    npm ci
    npm test
    npm run lint
"

echo "✅ Tests terminés pour Node.js ${NODE_VERSION}"
```
