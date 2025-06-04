 # Atelier 8 : Installation et configuration de plugins

**Objective :** Installer et configurer des plugins essentiels

## Étape 1 : Plugins via Docker

```dockerfile
# Dockerfile avec plugins pré-installés
FROM jenkins/jenkins:lts

# Installation des plugins essentiels
RUN jenkins-plugin-cli --plugins \
    git:latest \
    github:latest \
    nodejs:latest \
    docker-workflow:latest \
    pipeline-stage-view:latest \
    slack:latest \
    email-ext:latest \
    junit:latest \
    jacoco:latest \
    sonar:latest \
    owasp-markup-formatter:latest \
    build-timeout:latest \
    timestamper:latest \
    ws-cleanup:latest

# Configuration par défaut
COPY jenkins-config/ /usr/share/jenkins/ref/

# Scripts d'initialisation
COPY init-scripts/ /usr/share/jenkins/ref/init.groovy.d/
```

## Étape 2 : Configuration automatique

```groovy
// init-scripts/configure-plugins.groovy

import jenkins.model.*
import hudson.security.*
import org.jenkinsci.plugins.workflow.libs.*

def instance = Jenkins.getInstance()

// Configuration NodeJS
def nodeJs = instance.getDescriptor("jenkins.plugins.nodejs.tools.NodeJSInstallation")
def nodeInstallations = [
  new NodeJSInstallation("Node 16", null, [
    new NodeJSInstaller("16.20.0")
  ]),
  new NodeJSInstallation("Node 18", null, [
    new NodeJSInstaller("18.16.0")
  ])
]
nodeJs.setInstallations(nodeInstallations as NodeJSInstallation[])

// Configuration Docker
def docker = instance.getDescriptor("org.jenkinsci.plugins.docker.commons.tools.DockerTool")
def dockerInstallations = [
  new DockerInstallation("Docker", "/usr/bin/docker", [])
]
docker.setInstallations(dockerInstallations)

instance.save()
```
