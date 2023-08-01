@Library('jenkins-library') _

import groovy.json.JsonBuilder
import groovy.json.JsonSlurper
import java.io.File
System.setProperty("hudson.model.DirectoryBrowserSupport.CSP", "")

pipeline {
    agent {
        kubernetes(containerCall(imageName: ACR_NAME, credentialSecret: SECRET_NAME))
    }

    environment {
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
        stage('Prepare environment') {
            steps {
                echo '-=- prepare environment -=-'
                sh 'java -version'
                sh './mvnw --version'
                container('podman') {
                    sh 'podman --version'
                    sh "podman login $ACR_URL -u $AAD_SERVICE_PRINCIPAL_USR -p $AAD_SERVICE_PRINCIPAL_PSW"
                }
                container('aks-builder') {
                    sh "az login --service-principal --username $AAD_SERVICE_PRINCIPAL_USR --password $AAD_SERVICE_PRINCIPAL_PSW --tenant $AKS_TENANT"
                    sh "az aks get-credentials --resource-group $AKS_RESOURCE_GROUP --name $AKS_NAME"
                    sh "kubelogin convert-kubeconfig -l spn --client-id $AAD_SERVICE_PRINCIPAL_USR --client-secret $AAD_SERVICE_PRINCIPAL_PSW"
                    sh 'kubectl version'
                }
                script {
                    qualityGates = readYaml file: 'quality-gates.yaml'
                }
            }
        }

        stage('Compile') {
            steps {
                echo '-=- compiling project -=-'
                sh './mvnw compile'
            }
        }

        stage('Code inspection & quality gate') {
            steps {
                echo '-=- run code inspection & check quality gate -=-'
                withSonarQubeEnv('ci-sonarqube') {
                    sh "./mvnw clean compile sonar:sonar -Dsonar.projectKey=$APP_NAME-$BRANCH_MINUS -Dsonar.login=$SONAR_CREDENTIALS_USR -Dsonar.password=$SONAR_CREDENTIALS_PSW"
                }
            }
        }
    }
}
