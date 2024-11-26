# Installation et Configuration de Nexus Repository Manager avec Docker

## Objectifs
- Installer Nexus Repository Manager avec Docker
- Configurer des dépôts Maven
- Sécuriser l'accès à Nexus
- Mettre en place des dépôts pour différents types d'artefacts

## Prérequis
- Docker installé
- Docker Compose (optionnel mais recommandé)
- Port 8081 disponible
- Système Linux ou MacOS (recommandé)

## Étape 1 : Configuration Docker Compose

### Création du fichier `docker-compose.yml`
```yaml
version: '3'

services:
  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    restart: always
    volumes:
      - nexus-data:/nexus-data
    ports:
      - "8081:8081"
      - "8082:8082"  # Pour les dépôts Docker
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms512m -Xmx2048m 
      - NEXUS_SECURITY_RANDOMPASSWORD=false

volumes:
  nexus-data:
    driver: local
```

## Étape 2 : Lancement de Nexus

### Démarrer le conteneur
```bash
docker-compose up -d
```

### Vérifier le statut
```bash
docker logs -f nexus
```

## Étape 3 : Première Connexion et Configuration Initiale

### Récupérer le mot de passe administrateur
```bash
docker exec nexus cat /nexus-data/admin.password
```

### Configuration des Dépôts Maven

#### 1. Dépôts Proxy
- `maven-central`: Proxy vers Maven Central
- `maven-public`: Dépôt groupé incluant central et releases
- `maven-releases`: Dépôt pour les versions stables
- `maven-snapshots`: Dépôt pour les versions de développement

#### Script de Configuration via REST API
```bash
#!/bin/bash
# Configuration des dépôts Maven

# Credentials
USERNAME='admin'
PASSWORD='votre-mot-de-passe'
NEXUS_URL='http://localhost:8081'

# Fonction d'appel API
nexus_api_call() {
    curl -v -u $USERNAME:$PASSWORD \
         -H "Content-Type: application/json" \
         -H "accept: application/json" \
         "$@"
}

# Configuration des dépôts Maven Proxy
create_maven_proxy_repo() {
    local name=$1
    local remote_url=$2
    
    nexus_api_call \
        -X POST \
        "$NEXUS_URL/service/rest/v1/repositories/maven/proxy" \
        -d "{
            \"name\": \"$name\",
            \"online\": true,
            \"storage\": {
                \"blobStoreName\": \"default\",
                \"strictContentTypeValidation\": true
            },
            \"proxy\": {
                \"remoteUrl\": \"$remote_url\",
                \"contentMaxAge\": 1440,
                \"metadataMaxAge\": 1440
            },
            \"negativeCache\": {
                \"enabled\": true,
                \"timeToLive\": 1440
            },
            \"httpClient\": {
                \"blocked\": false,
                \"autoBlock\": true
            },
            \"routingRules\": null,
            \"maven\": {
                \"versionPolicy\": \"MIXED\",
                \"layoutPolicy\": \"PERMISSIVE\"
            }
        }"
}

# Création des dépôts
create_maven_proxy_repo maven-central https://repo1.maven.org/maven2/
create_maven_proxy_repo maven-google https://maven.google.com/
```

## Étape 4 : Configuration des Utilisateurs et Rôles

### Script de Création d'Utilisateurs
```bash
#!/bin/bash

NEXUS_URL='http://localhost:8081'
USERNAME='admin'
PASSWORD='votre-mot-de-passe'

# Création d'un utilisateur développeur
create_user() {
    local username=$1
    local password=$2
    local email=$3
    
    curl -v -u $USERNAME:$PASSWORD \
         -H "Content-Type: application/json" \
         -X POST \
         "$NEXUS_URL/service/rest/v1/security/users" \
         -d "{
             \"userId\": \"$username\",
             \"firstName\": \"$username\",
             \"lastName\": \"Développeur\",
             \"emailAddress\": \"$email\",
             \"password\": \"$password\",
             \"status\": \"active\",
             \"roles\": [\"nx-developer\"]
         }"
}

# Exemple de création d'un utilisateur
create_user "dev-user" "mot-de-passe-securise" "dev@exemple.com"
```

## Étape 5 : Configuration des Réglages de Sécurité

### Bonnes Pratiques
- Activer l'authentification
- Utiliser HTTPS
- Limiter les accès anonymes
- Configurer des rôles spécifiques

### Exemple de Configuration SSL
```bash
# Génération d'un certificat auto-signé
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout nexus.key -out nexus.crt
```

## Configuration Réseau et Firewall

### Règles de Pare-feu
```bash
# Autoriser les ports Nexus
sudo ufw allow 8081/tcp
sudo ufw allow 8082/tcp
```

## Script Complet d'Installation et Configuration

```bash
#!/bin/bash

# Mise à jour du système
sudo apt-get update
sudo apt-get upgrade -y

# Installation de Docker et Docker Compose
sudo apt-get install docker.io docker-compose -y

# Cloner le dépôt avec docker-compose.yml
git clone https://github.votre-depot/nexus-config.git
cd nexus-config

# Démarrer Nexus
docker-compose up -d

# Attendre le démarrage de Nexus
sleep 60

# Exécuter les scripts de configuration
bash configure-repos.sh
bash configure-users.sh
```

## Ressources Complémentaires
- Documentation Nexus : https://help.sonatype.com/repomanager3
- Docker Hub Nexus : https://hub.docker.com/r/sonatype/nexus3
```

Cet exercice propose une approche complète pour :
1. Installer Nexus avec Docker
2. Configurer les dépôts Maven
3. Gérer les utilisateurs et la sécurité
4. Mettre en place des bonnes pratiques

Voulez-vous que je développe un point en particulier ou que je vous aide à adapter ce guide à votre environnement spécifique ?