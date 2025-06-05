# ATELIER  : Création d'un Pipeline Moderne 

## Objectif
Créer un pipeline complet avec Jenkinsfile et l'explorer dans Blue Ocean.

## Étapes

1. **Préparation du Repository**
```bash
mkdir jenkins-pipeline-demo
cd jenkins-pipeline-demo
git init
```

2. **Création du Jenkinsfile**
```groovy
pipeline {
    agent any
    
    environment {
        APP_NAME = 'demo-app'
        VERSION = "${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Preparation') {
            steps {
                echo "Building ${APP_NAME} version ${VERSION}"
                sh 'echo "$(date): Starting build" > build.log'
            }
        }
        
        stage('Compile') {
            steps {
                sh 'echo "Compiling application..." >> build.log'
                sleep 2
            }
        }
        
        stage('Tests') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'echo "Running unit tests..." >> build.log'
                        sleep 3
                        sh 'echo "<?xml version=\\"1.0\\"?><testsuite tests=\\"5\\" failures=\\"0\\"><testcase name=\\"test1\\"/></testsuite>" > test-results.xml'
                    }
                }
                stage('Linting') {
                    steps {
                        sh 'echo "Running linter..." >> build.log'
                        sleep 2
                    }
                }
            }
        }
        
        stage('Package') {
            steps {
                sh 'tar czf ${APP_NAME}-${VERSION}.tar.gz build.log test-results.xml'
                archiveArtifacts artifacts: '*.tar.gz,*.xml'
            }
        }
    }
    
    post {
        always {
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'build.log',
                reportName: 'Build Log Report'
            ])
        }
    }
}
```

* Pousser le dépôt sur un dépôt public ou utiliser le dépôt : https://github.com/vanessakovalsky/jenkins-training (le fichier Jenkinsfile est dans le dossier corrige/atelier-pipeline/

3. **Configuration Jenkins**
- Créer un nouveau projet de type pipeline 
- Configurer la source Git et le chemin vers le jenkins file 
- Observer dans Blue Ocean

### Questions de Validation
- Comment visualiser l'exécution parallèle dans Blue Ocean ?
- Où retrouver les artifacts archivés ?
- Comment modifier le pipeline graphiquement ?
