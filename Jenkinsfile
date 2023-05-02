@Library('jenkins-library') _

import groovy.json.JsonBuilder
import groovy.json.JsonSlurper

pipeline {
    agent {
        kubernetes(containerCall(imageName: ACR_NAME, credentialSecret: SECRET_NAME))
    }

    environment {
        SELENIUM_HUB_URL = "http://selenium-hub:4444/wd/hub"
        BASE_URL = credentials('url-dependency')
        DEPENDENCY_API_KEY = credentials('dependency--api-key')
        PROJECT_NAME = 'prueba_dt'
        APP_NAME = 'deors-demos-java-pipeline'
        APP_VERSION = '1.0'
        APP_CONTEXT_ROOT = '/'
        APP_LISTENING_PORT = '8080'
        APP_JACOCO_PORT = '6300'
        IMAGE_PREFIX = 'deors'
        IMAGE_NAME = "$IMAGE_PREFIX/$APP_NAME"
        IMAGE_SNAPSHOT = "$IMAGE_NAME:snapshot-$BUILD_NUMBER"
        TEST_CONTAINER_NAME = "ephtest-$APP_NAME-$BUILD_NUMBER"
        BRANCH_SONAR = "$GIT_BRANCH"
        BRANCH_MINUS = BRANCH_SONAR.minus('origin/')

        // credentials & external systems
        AAD_SERVICE_PRINCIPAL = credentials('admins-rbac-sp')
        AKS_TENANT = credentials('aks-tenant')
        AKS_RESOURCE_GROUP = credentials('aks-resource-group')
        AKS_NAME = credentials('aks-name')
        //ACR_NAME = credentials('acr-name')
        ACR_URL = credentials('acr-url')
        // change this later
        ACR_PULL_CREDENTIAL = 'ndop-acr-credential-secret'
        SONAR_CREDENTIALS = credentials('sonar-new-credentials')
        SELENIUM_HUB_HOST = credentials('selenium-hub-host')
        SELENIUM_HUB_PORT = credentials('selenium-hub-port')
    }

    stages {
        
        stage('Web page performance analysis') {
            steps {
                echo '-=- execute web page performance analysis -=-'
                container('aks-builder') {
                    sh 'apt-get update'
                    sh 'apt-get install -y gnupg'
                    sh 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" | tee -a /etc/apt/sources.list.d/google.list'
                    sh 'curl -sL https://dl.google.com/linux/linux_signing_key.pub | apt-key add -'
                    sh 'curl -sL https://deb.nodesource.com/setup_10.x | bash -'
                    sh 'apt-get install -y nodejs google-chrome-stable'
                    sh 'npm install -g lighthouse@5.6.0'
                    sh "lighthouse http://$TEST_CONTAINER_NAME:$APP_LISTENING_PORT/$APP_CONTEXT_ROOT/hello --output=html --output=csv --chrome-flags=\"--headless --no-sandbox\""
                    archiveArtifacts artifacts: '*.report.html'
                    archiveArtifacts artifacts: '*.report.csv'
                    sh "lighthouse --view http://127.0.0.1:9001"
                }
            }
        }
    }

    post {
        always {
            echo '-=- stop test container and remove deployment -=-'
            container('aks-builder') {
                sh "kubectl delete pod $TEST_CONTAINER_NAME"
                sh "kubectl delete service $TEST_CONTAINER_NAME"
                sh "kubectl delete service $TEST_CONTAINER_NAME-jacoco"
            }
        }
    }
}
