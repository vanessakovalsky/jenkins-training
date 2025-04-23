# Exercice 2 - Construire son application et l'envoyer sur un dépôt

## Objectifs
Cet exercice a pour objectifs : 
* d'automatiser le build d'images docker dans une chaine de CI
* d'automatiser l'envoi d'images docker dans un dépôt d'image au sein d'une chaine CI

## Pré-requis
* Avoir un jenkins installé et un projet pipeline créé
* Avoir un dépôt Git avec au moins un Dockerfile
* Avoir un compte sur hub.docker.com

## Installation des plugins de Jenkins

* Dans l'interface de Jenkins, Aller dans Administrer Jenkins > Gestion des Plugins > Available Plugins
* Rechercher et sélectionner les plugins suivant :
    * Docker API Plugin
    * Docker Common Plugin
    * Docker Pipeline
    * Docker Plugin
    * docker-build-step
* Cliquer sur `Install without restart` et patienter que l'installation soit terminée

![](https://i0.wp.com/iot4beginners.com/wp-content/uploads/2022/08/image-22.png?resize=768%2C465&ssl=1)

## Ajouter les informations de connexions à docker dans Jenkins

* Dans l'interface aller dans Administrer Jenkins > Manage Credentials > System > Identifiants globaux (illimité) > Add Credentials
* Choisir :
    * type : Nom d'utilisateur et mot de passe
    * Scope : Global
    * Entrer votre nom d'utilisateur Docker et votre mot de passe
    * Choisir un ID pour ces identifiants (l'ID permet de les appeler dans les pipelines de Jenkins, c'est une sorte de variable)

![](https://iot4beginners.com/wp-content/uploads/2022/08/image-23-980x420.png)

## Ajouter le build à notre Pipeline

* Venir dans le projet créé auparavant
* Cliquer sur Configurer et entrer dans la section Pipeline le script suivant (adapter l'adresse du dépôt à votre propre dépôt, le nom de l'image en mettant votre username, et l'identifiant utilisé pour les informations de connexion à Docker) : 
```
pipeline {
    agent any
    stages {
        stage('Clone repository') {
            steps {
                git credentialsId: 'git', url: 'https://github.com/vanessakovalsky/python-api-handle-it'
            }
        }
        
        stage('Build image') {
            steps {
                script {
                    dockerImage = docker.build("vanessakovalsky/mypythonapp:latest","-f docker-app/python/Dockerfile .")
                }
            }
        }
        
         stage('Push image') {
            steps {
                script {
                    withDockerRegistry([ credentialsId: "dockerhubaccount", url: "" ]) {
                    dockerImage.push()
                    }
                }
            }
         }
    }
}                       
```
* Enregistrer et revenir sur le projet
* Lancer un build 
![](https://i0.wp.com/iot4beginners.com/wp-content/uploads/2022/08/image-27.png?resize=1024%2C558&ssl=1)
* Une fois le build terminé vous pouvez vérifier les logs de celui ci avec Console Output
* Puis vérifier sur le Hub docker que votre image a bien été push

## Pour aller plus loin 

* Dans certains cas vous aurez besoin d'un outil de Build comme Gradle, vous pouvez regarder sur ce dépôt pour découvrir l'utilisation de Gradle, du dépôt Nexus et leur intégration dans Jenkins : https://github.com/vanessakovalsky/continuous-integration-training 
