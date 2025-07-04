# ATELIER  : Déploiement Automatisé 

## Objectif
Créer un pipeline de déploiement multi-environnements avec validation.

## Pipeline de Déploiement

```groovy
pipeline {
    agent any
    
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Environment cible')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip les tests post-déploiement')
    }
    
    environment {
        APP_NAME = 'demo-app'
        VERSION = "${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Preparation') {
            steps {
                echo "Preparing deployment to ${params.ENVIRONMENT}"
                sh 'mkdir -p artifacts'
                sh 'echo "app-content" > artifacts/app.txt'
                sh 'echo "config-${ENVIRONMENT}" > artifacts/config.properties'
            }
        }
        
        stage('Pre-deployment Checks') {
            parallel {
                stage('Health Check Target') {
                    steps {
                        script {
                            def url = getEnvironmentUrl(params.ENVIRONMENT)
                            sh "echo 'Checking ${url}'"
                            // Simulation d'un health check
                            sh 'sleep 2'
                            echo "Target environment is healthy"
                        }
                    }
                }
                stage('Database Migration Check') {
                    steps {
                        sh 'echo "Checking database migrations..."'
                        sh 'sleep 1'
                        echo "Database is ready"
                    }
                }
                stage('Resource Validation') {
                    steps {
                        sh 'echo "Validating system resources..."'
                        sh 'sleep 1'
                        echo "Resources available"
                    }
                }
            }
        }
        
        stage('Deployment') {
            steps {
                script {
                    echo "Deploying ${APP_NAME} v${VERSION} to ${params.ENVIRONMENT}"
                    
                    // Simulation du déploiement
                    if (params.ENVIRONMENT == 'production') {
                        echo "Production deployment - Blue/Green strategy"
                        sh 'sleep 5'
                    } else {
                        echo "Non-prod deployment - Direct update"
                        sh 'sleep 3'
                    }
                    
                    sh 'tar czf deployment-${ENVIRONMENT}-${VERSION}.tar.gz artifacts/'
                    archiveArtifacts artifacts: 'deployment-*.tar.gz'
                }
            }
        }
        
        stage('Post-deployment Tests') {
            parallel {
                stage('Smoke Tests') {
                    steps {
                        sh 'echo "Running smoke tests..."'
                        sh 'sleep 3'
                        echo "Smoke tests passed"
                    }
                }
                stage('API Tests') {
                    steps {
                        sh 'echo "Running API tests..."'
                        sh 'sleep 4'
                        echo "API tests passed"
                    }
                }
            }
        }
        
        stage('Monitoring Setup') {
            steps {
                sh 'echo "Setting up monitoring for ${ENVIRONMENT}..."'
                sh 'echo "Deployment completed at $(date)" > deployment-${ENVIRONMENT}.log'
                archiveArtifacts artifacts: 'deployment-*.log'
            }
        }
    }
    
    post {
        success {
            echo "✅ Deployment to ${params.ENVIRONMENT} completed successfully"
            script {
                if (params.ENVIRONMENT == 'production') {
                    slackSend channel: '#deployments', 
                             message: "🚀 Production deployment successful: ${APP_NAME} v${VERSION}"
                }
            }
        }
        failure {
            echo "❌ Deployment to ${params.ENVIRONMENT} failed"
            script {
                // Rollback en cas d'échec
                if (params.ENVIRONMENT == 'production') {
                    sh 'rollback-production.sh'
                    slackSend channel: '#deployments', 
                             message: "🔴 Production deployment failed and rolled back: ${APP_NAME} v${VERSION}"
                }
            }
        }
    }
}

// Fonction helper pour obtenir l'URL de l'environnement
def getEnvironmentUrl(environment) {
    switch(environment) {
        case 'dev':
            return 'https://dev.example.com'
        case 'staging':
            return 'https://staging.example.com'
        case 'production':
            return 'https://app.example.com'
        default:
            return 'https://localhost'
    }
}
```
