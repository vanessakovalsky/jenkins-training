# 🛠️ ATELIER PRATIQUE : Installation Jenkins avec Docker
** Durée totale : 90 minutes**

## Objectif
Installer et configurer Jenkins dans un environnement Docker, puis effectuer la configuration initiale.

## Prérequis
- Docker installé sur votre machine
- Accès à Internet
- Port 8080 disponible

## Étape 1 : Préparation de l'environnement
** 5 minutes**

```bash
# Créer un réseau Docker pour Jenkins
docker network create jenkins

# Créer un volume pour persister les données
docker volume create jenkins-data
```

### Étape 2 : Lancement du container Jenkins
** 10 minutes**

```bash
# Lancer Jenkins avec Docker
docker run \
  --name jenkins-master \
  --restart=on-failure \
  --detach \
  --network jenkins \
  --publish 8080:8080 \
  --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  jenkins/jenkins:jdk21
```

## Étape 3 : Premier accès à Jenkins

1. **Ouvrir votre navigateur**
   - Aller sur `http://localhost:8080`

2. **Récupérer le mot de passe initial**
   ```bash
   # Afficher le mot de passe d'administration
   docker exec jenkins-master cat /var/jenkins_home/secrets/initialAdminPassword
   ```

3. **Interface de déverrouillage**
   ```
   ┌─────────────────────────────────────────┐
   │          Déverrouiller Jenkins          │
   │                                         │
   │  Pour s'assurer que Jenkins a été       │
   │  installé de façon sécurisée par un     │
   │  administrateur, un mot de passe a été  │
   │  écrit dans le log et ce fichier sur    │
   │  le serveur :                           │
   │                                         │
   │  /var/jenkins_home/secrets/             │
   │  initialAdminPassword                   │
   │                                         │
   │  ┌─────────────────────────────────────┐│
   │  │ [Mot de passe administrateur]       ││
   │  └─────────────────────────────────────┘│
   │                                         │
   │              [Continuer]                │
   └─────────────────────────────────────────┘
   ```

## Étape 4 : Installation des plugins
** 15 minutes (incluant temps d'installation)**

1. **Choisir "Install suggested plugins"**
   - Jenkins installera automatiquement les plugins essentiels

2. **Plugins installés automatiquement :**
   - Ant Plugin
   - Build Timeout
   - Credentials Binding
   - Email Extension
   - Git Plugin
   - Gradle Plugin
   - Pipeline
   - SSH Build Agents
   - Timestamper
   - Workspace Cleanup

## Étape 5 : Création du premier utilisateur administrateur
** 5 minutes**

```
┌─────────────────────────────────────────┐
│       Créer le premier utilisateur      │
│                                         │
│  Nom d'utilisateur : [admin]            │
│  Mot de passe :     [votre_mdp]         │
│  Confirmer :        [votre_mdp]         │
│  Nom complet :      [Admin Jenkins]     │
│  Email :           [admin@company.com]  │
│                                         │
│         [Sauvegarder et continuer]      │
└─────────────────────────────────────────┘
```

## Étape 6 : Configuration de l'URL d'instance
** 2 minutes**

```
Jenkins URL : http://localhost:8080/
```

## Étape 7 : Configuration des outils
** 20 minutes**

1. **Aller dans "Administrer Jenkins" > "Configuration globale des outils"**

2. **Configurer JDK :**
   ```
   Nom : JDK-11
   ☑ Installer automatiquement
   Version : openjdk-11.0.2
   ```

3. **Configurer Git :**
   ```bash
   # Dans le container Jenkins
   docker exec -it jenkins-master bash
   git config --global user.name "Jenkins"
   git config --global user.email "jenkins@company.com"
   ```

4. **Configurer Maven :**
   ```
   Nom : Maven-3.8
   ☑ Installer automatiquement
   Version : 3.8.6
   ```

## Étape 8 : Test de l'installation
** 15 minutes**

1. **Créer un job de test**
   - Cliquer sur "Nouvel élément"
   - Nom : `test-installation`
   - Type : "Projet free-style"

2. **Configuration du job**
   ```bash
   # Dans la section "Build"
   # Ajouter une étape "Exécuter un script shell"
   echo "Hello Jenkins!"
   echo "Java version:"
   java -version
   echo "Current date:"
   date
   ```

3. **Lancer le build**
   - Cliquer sur "Lancer un build"
   - Vérifier la console de sortie

### Étape 9 : Commandes utiles pour la gestion
** 8 minutes**

```bash
# Voir les logs Jenkins
docker logs jenkins-master

# Accéder au container
docker exec -it jenkins-master bash

# Arrêter Jenkins
docker stop jenkins-master

# Redémarrer Jenkins
docker start jenkins-master

# Sauvegarder les données Jenkins
docker run --rm \
  --volumes-from jenkins-master \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/jenkins-backup.tar.gz /var/jenkins_home
```

## Vérification de l'installation

### Checklist de validation

- [ ] Jenkins accessible sur http://localhost:8080
- [ ] Connexion administrateur fonctionnelle
- [ ] Plugins de base installés
- [ ] JDK configuré et fonctionnel
- [ ] Git disponible
- [ ] Maven configuré
- [ ] Job de test exécuté avec succès

### Résolution des problèmes courants

| Problème | Solution |
|----------|----------|
| Port 8080 occupé | Changer le port : `-p 8081:8080` |
| Permissions Docker | Ajouter l'utilisateur au groupe docker |
| Mémoire insuffisante | Augmenter `-e JAVA_OPTS="-Xmx2048m"` |
| Container ne démarre pas | Vérifier les logs avec `docker logs` |
