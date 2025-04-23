# Exercice 1 - Installer et configurer Jenkins

## Objectifs

Cet exercice a pour objectifs :
* d'avoir un Jenkins installé et configuré
* de découvrir l'interface et les concepts de Jenkins

## Pré-requis

* Avoir sur sa machine git, docker et docker-compose installés
* Créer un fork dans votre compte github du dépôt : https://github.com/vanessakovalsky/python-api-handle-it.git 
* Cloner le dépôt de cet exercice : git clone https://github.com/vanessakovalsky/jenkins-training 

## Installation de jenkins

* A partir du dossier cloné, se mettre dans le dossier docker et lancer la commande :
```
docker-compose up -d --build
```
* Cela permet d'avoir une image de jenkins qui contient Docker (qui nous sera utile pour les actions d'intégration et de déploiement continue) et de lancer un conteneur
* Une fois le conteneur lancé (vérifier qu'il est au statut up), se rendre à l'url : http://localhost:8080 pour accéder à l'interface de Jenkins

## Configurer le jenkins

* La première chose que vous demande Jenkins est le mot de passe pour déverrouiller jenkins
![](https://paper-attachments.dropbox.com/s_33CE5684927EB1F665F2EEF2A8A615DFA881F46F04918B588BABDF4D08ACF025_1645484899181_jenkins-getting-started.png)
* Pour récupérer ce mot de passe afficher les logs de votre conteneur, il doit se trouver dedans :
![](https://paper-attachments.dropbox.com/s_33CE5684927EB1F665F2EEF2A8A615DFA881F46F04918B588BABDF4D08ACF025_1645553155314_Screenshot+from+2022-02-22+19-05-37.png)
* Ensuite Jenkins vous propose d'installer les plugins suggérés ou de choisir les plugins à installer, laissez l'option par défaut d'installer les plugins suggérés :
![](https://paper-attachments.dropbox.com/s_33CE5684927EB1F665F2EEF2A8A615DFA881F46F04918B588BABDF4D08ACF025_1645661908679_plugins-installation.png)
* Puis Jenkins vous demande de créer un compte administrateur, remplir le formulaire et cliquer sur  `Save and continue`
![](https://paper-attachments.dropbox.com/s_33CE5684927EB1F665F2EEF2A8A615DFA881F46F04918B588BABDF4D08ACF025_1645717974971_Screenshot+from+2022-02-24+16-52-36.png)
* Votre Jenkins est maintenant prêt à être utilisé

## Découverte de Jenkins

* Lorsque vous arrivez sur le tableau de bord de Jenkins, vous pouvez créer un nouvel item
![](images/jenkins_new.png) 
* Il s'agit ici des éléments suivants :
    * Projet free-style : permet de construire des jobs jenkins avec l'ancienne façon de faire de jenkins, avec des tâches les unes après les autres
    * Pipeline : permet de construire des jobs jenkins avec des tâches en parallèle, c'est aujourd'hui la méthode recommandée d'utilisation de Jenkins
    * Projet multi-configuration : permet de gérer les gros projets avec de nombreux fichiers de configuration nécessaires
    * Dossier : permet de classer ces projets dans des dossiers
    * Organization folder : permet de créer des sous dossiers basés sur les branches d'un dépôt de code
    * Pipeline multibranche : permet de gérer plusieurs branches d'un dépôt de code au sein d'un même pipeline
* Créer un projet de type Pipeline en lui donnant un nom, en sélectionnant Pipeline et en cliquant sur OK
* Dans la rubrique Source code management, choisir Git et entrer l'URL du projet : https://github.com/vanessakovalsky/python-api-handle-it.git (mettre l'URL de votre propre dépôt)
* Cliquer sur Sauver
* Vous arrivez sur la page du Job
* Vous pouvez exécuter le job en cliquant sur Lancer un build
* Le job se lance, cliquez dessus pour avoir des informations
* En allant dans Console Output, que voyez-vous ? 


## Configurer Github pour déclencher un build à chaque commit

### Si votre projet jenkins a une URL publique : 
* Vous avez besoin d'un token d'API de Jenkins pour configurer le webhook dans github
* Sur jenkins, cliquer sur votre nom en haut à droite
* Puis sur Configure dans le menu
* Dans la section API Token, cliquer sur Add new token, et copier le token généré

### Si votre projet jenkins a une URL locale non exposée sur internet :

* Si votre jenkins est en local il faut en plus installer et utiliser un relay qui se permettra à Jenkins et à Github de communiquer via une URL publique. Pour cela voici les étapes à suivre : 
    * Installer l'outil CLI en fonction de votre OS : https://webhookrelay.com/v1/installation/cli 
    * Se connecter (avec votre compte github) sur https://my.webhookrelay.com/ 
    * Générer un token d'identification sur la page Token (une fois connecté dans le menu de gauche Access tokens > + Generate token) : https://my.webhookrelay.com/tokens 
    * Lors de la creation du token, webhook relay vous donne une commande de connexion de la forme
    ```
    relay login -k [key] -s [secret]
    ```
    * Copiez cette commande et exécutez-la dans un terminal pour pouvoir vous connecter à webhook relay
    * Puis lançons le relay entre votre jenkins local et webhook relay avec la commande :
    ```
    relay forward --bucket github-jenkins http://localhost:8080/github-webhook/
    ```
    * Vous obtenez alors une réponse de ce type-là, notez l'URL qui se trouve dans la réponse :
    ```
    Forwarding:
    https://my.webhookrelay.com/v1/webhooks/6edf55c7-e774-46f8-a058-f4d9c527a6a7 -> http://localhost:8080/github-webhook/
    Starting webhook relay agent...
    1.511438424864371e+09    info    webhook relay ready...    {"host": "api.webhookrelay.com:8080"}
    ```
    * Vous avez l'URL nécessaire pour la suite

### Dans tous les cas il faut configurer github et jenkins comme suit une fois l'URL obtenue

* Aller sur votre projet Github
* Dans les paramètres, choisir Webhook
* Cliquer sur Add webhook :
* * Dans payload URL, entrer l'adresse de Jenkins sous la forme
    * Pour les url publiques: http://[login]:[APITOKEN]@[URLduJenkins.com:8080]/github-webhook
    * Pour les URL locales : https://[id].hooks.webrelay.com  (URL à récupérer lors du lancement de la commande relay forward, qui doit rester démarrée pour faire vos tests (pas de Ctrl+C))
* Enregistrer
* Côté Jenkins, aller dans le job
* Dans la rubrique Build Triggers, cochez la case : GitHub hook trigger for GITScm polling 




-> Votre job est configuré pour être lancé à chaque push sur la branche master dans votre dépôt Github
/!\ Afin que le lien se fasse, il faut configurer un pipeline qui fera un git clone au moins, et le lancer une première fois
