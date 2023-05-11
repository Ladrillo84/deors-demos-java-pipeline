@Library('jenkins-library') _

import groovy.json.JsonBuilder
import groovy.json.JsonSlurper
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

        stage('Mutation tests') {
            steps {
                echo '-=- execute mutation tests -=-'
                sh './mvnw org.pitest:pitest-maven:mutationCoverage'
            }
        }

        stage('Generate BOM') {
            steps {
                sh './mvnw org.cyclonedx:cyclonedx-maven-plugin:makeBom'
            }
        }

        stage('Dependency Tracker') {
            steps {
                dependencyTrackPublisher artifact: 'target/bom.xml',
                    projectName: env.APP_NAME,
                    projectVersion: env.BUILD_NUMBER,
                    synchronous: true,
                    failedTotalCritical:    qualityGates.security.dependencies.critical.failed,
                    unstableTotalCritical:  qualityGates.security.dependencies.critical.unstable,
                    failedTotalHigh:        qualityGates.security.dependencies.high.failed,
                    unstableTotalHigh:      qualityGates.security.dependencies.high.unstable,
                    failedTotalMedium:      qualityGates.security.dependencies.medium.failed,
                    unstableTotalMedium:    qualityGates.security.dependencies.medium.unstable
            }
        }

        /*stage('Software composition analysis') {
            steps {
                echo '-=- run software composition analysis -=-'
                sh './mvnw dependency-check:check'
                dependencyCheckPublisher(
                    failedTotalCritical: qualityGates.security.dependencies.critical.failed,
                    unstableTotalCritical: qualityGates.security.dependencies.critical.unstable,
                    failedTotalHigh: qualityGates.security.dependencies.high.failed,
                    unstableTotalHigh: qualityGates.security.dependencies.high.unstable,
                    failedTotalMedium: qualityGates.security.dependencies.medium.failed,
                    unstableTotalMedium: qualityGates.security.dependencies.medium.unstable)
                script {
                    if (currentBuild.result == 'FAILURE') {
                        error('Dependency vulnerabilities exceed the configured threshold')
                    }
                }
            }
        }*/

        stage('Package') {
            steps {
                echo '-=- packaging project -=-'
                sh './mvnw package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Build & push container image') {
            steps {
                echo '-=- build & push container image -=-'
                container('podman') {
                    sh "podman build -t $IMAGE_SNAPSHOT ."
                    sh "podman tag $IMAGE_SNAPSHOT $ACR_URL/$IMAGE_SNAPSHOT"
                    sh "podman push $ACR_URL/$IMAGE_SNAPSHOT"
                }
            }
        }

        stage('Run container image') {
            steps {
                echo '-=- run container image -=-'
                container('aks-builder') {
                    sh "kubectl run $TEST_CONTAINER_NAME --image=$ACR_URL/$IMAGE_SNAPSHOT --env=JAVA_OPTS=-javaagent:/jacocoagent.jar=output=tcpserver,address=*,port=$APP_JACOCO_PORT --port=$APP_LISTENING_PORT --overrides='{\"apiVersion\": \"v1\", \"spec\": {\"imagePullSecrets\": [{\"name\": \"$ACR_PULL_CREDENTIAL\"}]}}'"
                    sh "kubectl expose pod $TEST_CONTAINER_NAME --port=$APP_LISTENING_PORT"
                    sh "kubectl expose pod $TEST_CONTAINER_NAME --port=$APP_JACOCO_PORT --name=$TEST_CONTAINER_NAME-jacoco"
                }
            }
        }

        stage('Integration tests') {
             steps {
                 echo '-=- execute integration tests -=-'
                 sh "curl --retry 10 --retry-connrefused --connect-timeout 5 --max-time 5 http://$TEST_CONTAINER_NAME:$APP_LISTENING_PORT" + "$APP_CONTEXT_ROOT/actuator/health".replace('//', '/')
                 sh "./mvnw failsafe:integration-test failsafe:verify -DargLine=-Dtest.selenium.hub.url=http://$SELENIUM_HUB_HOST:$SELENIUM_HUB_PORT/wd/hub -Dtest.target.server.url=http://$TEST_CONTAINER_NAME:$APP_LISTENING_PORT" + "$APP_CONTEXT_ROOT/".replace('//', '/')
                 sh "java -jar target/dependency/jacococli.jar dump --address $TEST_CONTAINER_NAME-jacoco --port $APP_JACOCO_PORT --destfile target/jacoco-it.exec"
                 sh 'mkdir -p target/site/jacoco-it'
                 sh 'java -jar target/dependency/jacococli.jar report target/jacoco-it.exec --classfiles target/classes --xml target/site/jacoco-it/jacoco.xml'
                 junit 'target/failsafe-reports/*.xml'
                 jacoco execPattern: 'target/jacoco-it.exec'
             }
         }

        stage('Performance tests') {
            steps {
                echo '-=- execute performance tests -=-'
                sh "curl --retry 10 --retry-connrefused --connect-timeout 5 --max-time 5 http://$TEST_CONTAINER_NAME:$APP_LISTENING_PORT" + "$APP_CONTEXT_ROOT/actuator/health".replace('//', '/')
                sh "./mvnw jmeter:configure@configuration jmeter:jmeter jmeter:results -Djmeter.target.host=$TEST_CONTAINER_NAME -Djmeter.target.port=$APP_LISTENING_PORT -Djmeter.target.root=$APP_CONTEXT_ROOT"
                perfReport(
                    sourceDataFiles: 'target/jmeter/results/*.csv',
                    errorUnstableThreshold: qualityGates.performance.throughput.error.unstable,
                    errorFailedThreshold: qualityGates.performance.throughput.error.failed,
                    errorUnstableResponseTimeThreshold: qualityGates.performance.throughput.response.unstable)
            }
        }

        
        stage('Web page performance analysis') {
            steps {
                echo '-=- execute web page performance analysis -=-'
                script {
                    // def qualityGatesLighthouse = readYaml file: 'Lighthouse-quality-gates.yaml'
                    // println("aki " + qualityGatesLighthouse)
                    String pathFile = "${pwd()}/Lighthouse-quality-gates.yaml";
                    println(pathFile)
                    def lighthousejob = build job: "lightHouseUsingLib",  parameters: [string(name: 'TEST_CONTAINER_NAME', value: "$env.TEST_CONTAINER_NAME"),
                                                       string(name: 'APP_CONTEXT_ROOT', value: "$env.APP_CONTEXT_ROOT"),
                                                       string(name: 'APP_LISTENING_PORT', value: String.valueOf("$env.APP_LISTENING_PORT")),
                                                       string(name: 'GIT_REPO_URL', value: gitUtility.getGitUrlRepositoryUnderPipeline()),
                                                       string(name: 'BRANCH_NAME', value: gitUtility.getGitBranchUnderPipeline()),
                                                       file(name: 'QUALITY_GATES', value: pathFile)]
                    copyArtifacts(projectName: "pipelineLighthouse", selector: specific("${lighthousejob.number}"))                    
                    lighthouseReport('./report.json')
                }
            }
        }
                     
        stage('Promote container image') {
            steps {
                echo '-=- promote container image -=-'
                container('podman') {
                    // use latest or a non-snapshot tag to deploy to production
                    sh "podman tag $IMAGE_SNAPSHOT $ACR_URL/$IMAGE_NAME:$APP_VERSION"
                    sh "podman push $ACR_URL/$IMAGE_NAME:$APP_VERSION"
                    sh "podman tag $IMAGE_SNAPSHOT $ACR_URL/$IMAGE_NAME:latest"
                    sh "podman push $ACR_URL/$IMAGE_NAME:latest"
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
