# Exercice : Installation et Configuration de Jenkins avec Docker

## Objectifs Pédagogiques
- Installer Docker sur un système Linux
- Télécharger et configurer une image Jenkins officielle
- Mettre en place un environnement Jenkins sécurisé
- Comprendre les étapes de base de l'intégration continue avec Jenkins et Docker

## Prérequis
- Un système Linux (Ubuntu 20.04 LTS recommandé)
- Droits administrateur (sudo)
- Connexion Internet
- Connaissances de base en ligne de commande

## Étape 1 : Préparation de l'environnement système

### 1.1 Mise à jour du système
```bash
sudo apt-get update
sudo apt-get upgrade -y
```

### 1.2 Installation des prérequis pour Docker
```bash
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```

## Étape 2 : Installation de Docker

### 2.1 Ajout du dépôt Docker officiel
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

### 2.2 Installation de Docker
```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```

### 2.3 Vérification de l'installation Docker
```bash
sudo docker --version
sudo systemctl status docker
```

### 2.4 Configuration des permissions utilisateur
```bash
sudo usermod -aG docker $USER
newgrp docker
```

## Étape 3 : Téléchargement et Configuration de Jenkins

### 3.1 Création d'un volume persistant pour Jenkins
```bash
docker volume create jenkins-data
```

### 3.2 Téléchargement de l'image Jenkins
```bash
docker pull jenkins/jenkins:lts
```

### 3.3 Lancement du conteneur Jenkins
```bash
docker run -d \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins-data:/var/jenkins_home \
  --name jenkins \
  jenkins/jenkins:lts
```

## Étape 4 : Configuration Initiale de Jenkins

### 4.1 Récupération du mot de passe initial
```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### 4.2 Configuration du Wizard Jenkins
1. Ouvrir un navigateur à l'adresse : `http://localhost:8080`
2. Entrer le mot de passe initial récupéré
3. Choisir l'installation des plugins recommandés
4. Créer un utilisateur administrateur

## Étape 5 : Configuration Avancée avec Docker

### 5.1 Installation du plugin Docker
1. Aller dans "Manage Jenkins" > "Manage Plugins"
2. Rechercher et installer le plugin "Docker Pipeline"

### 5.2 Configuration de l'agent Docker
1. "Manage Jenkins" > "Manage Nodes and Clouds"
2. "Configure Clouds" > "Add a new cloud" > "Docker"
3. Configurer le Docker Host : `unix:///var/run/docker.sock`


## Bonnes Pratiques
- Toujours utiliser des volumes pour la persistance
- Sécuriser l'accès à Jenkins avec HTTPS
- Mettre à jour régulièrement Jenkins et ses plugins
- Utiliser des credentials sécurisés

## Dépannage
- Vérifier les logs Docker : `docker logs jenkins`
- Redémarrer le conteneur : `docker restart jenkins`
- Vérifier les permissions et les volumes

## Ressources Complémentaires
- Documentation Docker : https://docs.docker.com
- Documentation Jenkins : https://www.jenkins.io/doc/