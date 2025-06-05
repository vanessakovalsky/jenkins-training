# ATELIER : Pipeline avec Docker et Kubernetes

### Objectif
Créer un pipeline utilisant Docker pour les builds et Kubernetes pour le déploiement.


## Préparation

* créer un projet Pipeline
* Installer le plugin Docker pipeline 

### Pipeline Complet

```groovy
pipeline {
    agent none
    
    environment {
        DOCKER_REGISTRY = 'localhost:5000'
        APP_NAME = 'demo-k8s-app'
        VERSION = "${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Build with Docker') {
            agent {
                docker {
                    image 'maven:3.8-openjdk-11-slim'
                    args '-v maven-cache:/root/.m2'
                }
            }
            steps {
                echo "Building application with Maven in Docker"
                sh '''
                    echo 'Creating mock Java application...'
                    mkdir -p src/main/java/com/example
                    cat > src/main/java/com/example/App.java << 'EOF'
public class App {
    public static void main(String[] args) {
        System.out.println("Hello from Dockerized Build!");
    }
}
EOF
                    
                    cat > pom.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>demo-app</artifactId>
    <version>1.0.0</version>
    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
    </properties>
</project>
EOF
                '''
                
                sh 'mvn clean compile package'
                
                // Créer un Dockerfile
                sh '''
                    cat > Dockerfile << 'EOF'
FROM openjdk:11-jre-slim
COPY target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
EOF
                '''
                
                stash includes: 'target/*.jar, Dockerfile', name: 'build-artifacts'
            }
        }
        
        stage('Multi-Container Testing') {
            agent any
            steps {
                unstash 'build-artifacts'
                
                script {
                    // Démarrer une stack de test avec docker-compose
                    writeFile file: 'docker-compose.test.yml', text: '''
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - ENV=test
    depends_on:
      - redis
      - postgres
  
  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
  
  postgres:
    image: postgres:13-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - "5432:5432"
  
  test-runner:
    image: curlimages/curl:latest
    depends_on:
      - app
    command: |
      sh -c "
        sleep 10
        echo 'Running health check...'
        curl -f http://app:8080/health || echo 'Health check would run here'
        echo 'Tests completed'
      "
'''
                    
                    try {
                        sh 'docker-compose -f docker-compose.test.yml up --build --abort-on-container-exit'
                    } finally {
                        sh 'docker-compose -f docker-compose.test.yml down -v || true'
                    }
                }
            }
        }
        
        stage('Container Registry Push') {
            agent any
            steps {
                unstash 'build-artifacts'
                
                script {
                    // Build et push de l'image
                    def image = docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}")
                    
                    // Simulation du push (dans un vrai environnement)
                    echo "Would push image: ${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}"
                    // image.push()
                    // image.push('latest')
                }
            }
        }
        
        stage('Kubernetes Deployment Simulation') {
            agent {
                docker {
                    image 'bitnami/kubectl:latest'
                    args '--entrypoint=""'
                }
            }
            steps {
                script {
                    // Créer des manifestes Kubernetes
                    writeFile file: 'k8s-deployment.yaml', text: """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  labels:
    app: ${APP_NAME}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      containers:
      - name: ${APP_NAME}
        image: ${DOCKER_REGISTRY}/${APP_NAME}:${VERSION}
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: "${VERSION}"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-service
spec:
  selector:
    app: ${APP_NAME}
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
"""
                    
                    // Simulation du déploiement (dans un vrai environnement avec kubectl configuré)
                    echo "Kubernetes manifests created:"
                    sh 'cat k8s-deployment.yaml'
                    
                    echo "Would execute:"
                    echo "kubectl apply -f k8s-deployment.yaml"
                    echo "kubectl rollout status deployment/${APP_NAME}"
                    echo "kubectl get pods -l app=${APP_NAME}"
                }
                
                archiveArtifacts artifacts: 'k8s-deployment.yaml'
            }
        }
        
        stage('Monitoring Setup') {
            agent {
                docker {
                    image 'alpine:latest'
                }
            }
            steps {
                sh '''
                    echo "Setting up monitoring for ${APP_NAME}..."
                    echo "Version: ${VERSION}"
                    echo "Registry: ${DOCKER_REGISTRY}"
                    
                    # Simulation de la configuration de monitoring
                    cat > monitoring-config.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: ${APP_NAME}-monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: '${APP_NAME}'
      static_configs:
      - targets: ['${APP_NAME}-service:8080']
EOF
                    
                    echo "Monitoring configuration created"
                    cat monitoring-config.yaml
                '''
                
                archiveArtifacts artifacts: 'monitoring-config.yaml'
            }
        }
    }
    
    post {
        always {
            node('any') {
                // Nettoyage des images Docker locales
                sh '''
                    docker image prune -f || true
                    docker system prune -f || true
                '''
            }
        }
        success {
            echo "🎉 Pipeline completed successfully!"
            echo "Application ${APP_NAME}:${VERSION} is ready for deployment"
        }
        failure {
            echo "❌ Pipeline failed"
            echo "Check logs and container status"
        }
    }
}
```
