# Exercice 5 - Afficher les rapports HTML dans jenkins

## Objectifs
Cet exercice a pour objectifs :
* d'ajouter à Jenkins un moyen de récupérer les rapports HTML et de les afficher

## Pré-requis
* Avoir suivi les exercices précédents

## Installation du plugin

* Pour afficher les rapports HTML nous allons utiliser un plugin
* Rechercher et installer dans la gestion des plugins, le plugin HTML Publisher

## Utilisation dans un pipeline

* Génération du rapport html avec pylint, remplacer l'étape(step) qui execute pylint par les lignes suivantes
```
 steps {
                sh 'mkdir -p app/reports/pylint; pylint --output-format json --recursive yes --exit-zero app > app/reports/pylint/report.json;pylint-json2html -o app/reports/pylint/report.html app/reports/pylint/report.json'
            }
        }
```
* Ajouter au pipeline de notre projet l'étape suivante
```
        stage('Publish HTML'){
            steps {
                publishHTML (target : [allowMissing: false,
                     alwaysLinkToLastBuild: true,
                     keepAll: true,
                     reportDir: 'app/reports/pylint/',
                     reportFiles: 'report.html',
                     reportName: 'My Pylint Reports',
                     reportTitles: 'The Report'])
            }
        }
```
* Enregistrer votre pipeline et lancer un build
* Vous devriez à la fin du build avoir accès dans le menu à un item intitulé My Pylint Reports qui affiche le rapport de pylint

## Ajouter les autres rapports

* Pour chaque outils d'analyse qualité et de tests que vous utilisez, générer un rapport HTML et publié le dans Jenkins avec HTML Publisher


