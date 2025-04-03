# Exercice 4 - Déploiement continu de l'application

## Objectifs 

Cet exercice a pour objectifs :
* d'ajouter du déploiement continu à sa chaine CI/CD
* de lancer l'application dans un conteneur

## Pré-requis
* Avoir suivi les exercices précédents

## Ajouter une étape de déploiement avec le lancement d'un conteneur

* Une fois notre projet intégré et buildé, il est temps de le déployer
* Comme nous avons utilisé docker, nous allons lancer l'application dans un conteneur docker
* Pour cela ajoutez l'étape suivante à votre pipeline :
```
 stage('Run Docker container on Jenkins Agent') {
             
            steps 
   {
                sh "docker run -d -p 5000:5000 vanessakovalsky/mypythonapp"
 
            }
        }
```

## Pour aller plus loin
* Il est également possible d'automatiser la création de l'infrastructure avec terraform ou un outil de creation d'infrastructure cloud qui seront appelés par Jenkins
* De la même façon les conteneurs peuvent être déployés dans des orchestrateurs de type Kubernetes, GKE, ECS, OpenShift 
* Un exemple ici : https://cursus-janvier2020.uptime-formation.fr/04-jenkins/tp1_jenkins_simple/ 
