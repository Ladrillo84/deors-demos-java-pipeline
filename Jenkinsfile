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
        stage('call antoher pipeline') {
            steps {
                script {
                    prueba = build job: "pipelineLighthouse",  parameters: [string(name: 'TEST_CONTAINER_NAME', value: "$env.TEST_CONTAINER_NAME"), 
                                                                              string(name: 'APP_CONTEXT_ROOT', value: "$env.APP_CONTEXT_ROOT"),
                                                                              string(name: 'APP_LISTENING_PORT', value: "$env.APP_LISTENING_PORT")]
                }
            }
        }
    }
}
