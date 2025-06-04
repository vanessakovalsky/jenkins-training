 # Atelier : Configuration des notifications

**Objective :** Configurer un système de notifications multi-canal

## Étape 1 : Configuration Email

```groovy
// Configuration Email via Groovy script
import jenkins.model.*
import hudson.tasks.Mailer

def instance = Jenkins.getInstance()
def mailServer = instance.getDescriptor("hudson.tasks.Mailer")

mailServer.setSmtpHost("smtp.gmail.com")
mailServer.setUseSsl(true)
mailServer.setSmtpPort("465")
mailServer.setSmtpAuth("jenkins@votre-domaine.com", "password")
mailServer.setCharset("UTF-8")

instance.save()
```

## Étape 2 : Configuration Slack

```bash
# Installation du plugin Slack
curl -X POST http://jenkins:8080/pluginManager/installNecessaryPlugins \
  -H "Jenkins-Crumb: ${CRUMB}" \
  -d "plugin.slack.default=on"

# Configuration via webhook
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
```

```yaml
# Configuration Slack dans Jenkins
Slack Notifications:
  Workspace: "votre-workspace"
  Credential: "slack-token"
  Default channel: "#ci-cd"
  Custom message: |
    🚀 *${JOB_NAME}* - Build #${BUILD_NUMBER}
    📊 Status: ${BUILD_STATUS}
    ⏱️ Duration: ${BUILD_DURATION}
    🔗 <${BUILD_URL}|View Build>
    
    📝 Changes:
    ${CHANGES_SINCE_LAST_SUCCESS}
```

## Étape 3 : Script de notification personnalisé

```bash
#!/bin/bash
# notification-script.sh

JOB_NAME="${JOB_NAME}"
BUILD_NUMBER="${BUILD_NUMBER}"
BUILD_STATUS="${BUILD_STATUS}"
BUILD_URL="${BUILD_URL}"
GIT_COMMIT="${GIT_COMMIT}"

# Function pour envoyer à Slack
send_slack_notification() {
    local status=$1
    local color=""
    local emoji=""
    
    case $status in
        "SUCCESS")
            color="good"
            emoji="✅"
            ;;
        "FAILURE")
            color="danger"
            emoji="❌"
            ;;
        "UNSTABLE")
            color="warning"
            emoji="⚠️"
            ;;
    esac
    
    curl -X POST -H 'Content-type: application/json' \
        --data "{
            \"attachments\": [{
                \"color\": \"${color}\",
                \"title\": \"${emoji} ${JOB_NAME} - Build #${BUILD_NUMBER}\",
                \"title_link\": \"${BUILD_URL}\",
                \"fields\": [
                    {\"title\": \"Status\", \"value\": \"${BUILD_STATUS}\", \"short\": true},
                    {\"title\": \"Branch\", \"value\": \"${GIT_BRANCH}\", \"short\": true},
                    {\"title\": \"Commit\", \"value\": \"${GIT_COMMIT:0:8}\", \"short\": true},
                    {\"title\": \"Duration\", \"value\": \"${BUILD_DURATION}\", \"short\": true}
                ],
                \"footer\": \"Jenkins CI\",
                \"ts\": $(date +%s)
            }]
        }" \
        ${SLACK_WEBHOOK_URL}
}

# Function pour envoyer un email
send_email_notification() {
    local status=$1
    local subject="[Jenkins] ${JOB_NAME} - Build #${BUILD_NUMBER} - ${status}"
    
    cat > email_body.html << EOF
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; }
        .header { background-color: #f0f0f0; padding: 20px; }
        .content { padding: 20px; }
        .success { color: #28a745; }
        .failure { color: #dc3545; }
        .unstable { color: #ffc107; }
    </style>
</head>
<body>
    <div class="header">
        <h2 class="${status,,}">Build ${status} - ${JOB_NAME}</h2>
    </div>
    <div class="content">
        <p><strong>Build Number:</strong> ${BUILD_NUMBER}</p>
        <p><strong>Build URL:</strong> <a href="${BUILD_URL}">${BUILD_URL}</a></p>
        <p><strong>Git Commit:</strong> ${GIT_COMMIT}</p>
        <p><strong>Git Branch:</strong> ${GIT_BRANCH}</p>
        <p><strong>Build Duration:</strong> ${BUILD_DURATION}</p>
        
        <h3>Recent Changes:</h3>
        <pre>${CHANGES_SINCE_LAST_SUCCESS}</pre>
        
        <h3>Console Output (Last 50 lines):</h3>
        <pre>$(tail -n 50 ${BUILD_LOG_FILE})</pre>
    </div>
</body>
</html>
EOF

    # Envoi de l'email
    mail -s "${subject}" \
         -a "Content-Type: text/html" \
         "${EMAIL_RECIPIENTS}" < email_body.html
}

# Function pour Teams
send_teams_notification() {
    local status=$1
    local theme_color=""
    
    case $status in
        "SUCCESS") theme_color="00FF00" ;;
        "FAILURE") theme_color="FF0000" ;;
        "UNSTABLE") theme_color="FFA500" ;;
    esac
    
    curl -H "Content-Type: application/json" \
         -d "{
            \"@type\": \"MessageCard\",
            \"@context\": \"https://schema.org/extensions\",
            \"summary\": \"Jenkins Build ${status}\",
            \"themeColor\": \"${theme_color}\",
            \"sections\": [{
                \"activityTitle\": \"Jenkins Build ${status}\",
                \"activitySubtitle\": \"${JOB_NAME} - Build #${BUILD_NUMBER}\",
                \"facts\": [
                    {\"name\": \"Status\", \"value\": \"${BUILD_STATUS}\"},
                    {\"name\": \"Branch\", \"value\": \"${GIT_BRANCH}\"},
                    {\"name\": \"Commit\", \"value\": \"${GIT_COMMIT:0:8}\"},
                    {\"name\": \"Duration\", \"value\": \"${BUILD_DURATION}\"}
                ],
                \"potentialAction\": [{
                    \"@type\": \"OpenUri\",
                    \"name\": \"View Build\",
                    \"targets\": [{\"os\": \"default\", \"uri\": \"${BUILD_URL}\"}]
                }]
            }]
         }" \
         ${TEAMS_WEBHOOK_URL}
}

# Envoi des notifications selon le statut
echo "📢 Envoi des notifications pour ${BUILD_STATUS}"

send_slack_notification "${BUILD_STATUS}"
send_email_notification "${BUILD_STATUS}"
send_teams_notification "${BUILD_STATUS}"

echo "✅ Notifications envoyées"
```
