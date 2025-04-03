# Exercice 3 - Ajouter l'exécution des tests à sa chaine de CI

## Objectifs
Cet exercice a pour objectifs
* de faire exécuter de l'analyse qualité à Jenkins
* de faire exécuter les tests unitaire de l'application à Jenkins

## Pré-requis
* Avoir fait l'exercice 1 et l'exercice 2
* Avoir une image de conteneur avec un outil d'analyse qualité
* Avoir une image de conteneur permettant l'exécution de tests unitaires
* Avoir des tests unitaires à exécuter

## Ajout de l'analyse qualité avec Pylint dans notre CI

* Revenir dans son projet et rajouter l'étape suivante à notre pipeline afin de builder l'image d'analyse qualité et d'exécuter le conteneur associé
```
         stage('Build Pylint image') {
            steps {
                script {
                    dockerImage = docker.build("vanessakovalsky/mypylint:latest","-f docker-test/pylint/Dockerfile docker-test/pylint/")
                }
            }
        }
        
        stage('Push pylint image') {
            steps {
                script {
                    withDockerRegistry([ credentialsId: "dockerhubaccount", url: "" ]) {
                    dockerImage.push()
                    }
                }
            }
        }
        
        stage('Pylint') { 
            agent {
                docker {
                    image 'vanessakovalsky/mypylint'
                    args '-v ${PWD}:/app'
                    reuseNode true
                }
            }
            steps {
                sh 'pylint --recursive yes --exit-zero app'
            }
        }
```
* Ici quelques remarques sur la syntaxe :
    * agent : définit l'environnement d'exécution de l'étape
    * reuseNode true : permet d'utiliser l'environnement des étapes précédentes (le contenu du git clone)
    * args : permet de donner des options au lancement d'un conteneur, ici un montage dans le dossier qui nous intéresse
    * Pour pylint : pour ne pas empêcher la suite l'option --exit-zero : permet de renvoyer un code de sortie de 0, soit de valider l'étape, de sorte à ce que s'il y a des erreurs de remontées, le reste des étapes puisse s'exécuter
* Sauvegarder la configuration et lancer un build

## Ajouter l'exécution des tests unitaire

* De la même façon que l'on a ajouté l'execution de pylint, ajouter les étapes suivantes au pipeline :
    * build de l'image pour les test
    * push de l'image pour les tests
    * exécution du conteneur qui exécute les tests

## Pour aller plus loin

* La documentation de la syntaxe de pipeline : https://www.jenkins.io/doc/book/pipeline/syntax/ 
* Il existe de nombreuses options pour les étapes de pipeline, vous pouvez vous aider de la documentation pour ajouter des étapes à votre build (execution de robot framework, autres outils d'analyse qualité ...)
* Des exemples d'étapes sont disponibles ici : https://www.jenkins.io/doc/pipeline/examples/ 
