pipeline {
    agent any

    environment {
        APP_NAME = 'Teckvisuals-sample-project'
        RELEASE = '1.0.0'
        DOCKER_USER = 'teckvisuals'
        GITHUB_URL = 'https://github.com/Teckvisuals/register-app.git'
        JENKINS_SERVER_URL = 'http://35.183.51.38:8080'
    }

    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from SCM') {
            steps {
                checkout([$class: 'GitSCM', 
                    branches: [[name: 'main']], 
                    userRemoteConfigs: [[url: GITHUB_URL, credentialsId: 'Teckvisuals-Git-Cred']]
                ])
            }
        }

        stage('Build Application') {
            steps {
                dir('maven-code/spring-boot-backend') {
                   sh 'mvn clean package'
               } 
           }
        }

        stage('Test Application') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Sonarqube Analysis') {
            steps {
                script {
                    def scannerHome = tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                    withSonarQubeEnv(credentialsId: 'Sonar-Jenks-Cred', installationName: scannerHome) {
                        sh 'mvn clean compile sonar:sonar'
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
                    def IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
                    sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Login to DockerHub') {
            steps {
                script {
                    def DOCKERHUB_CREDENTIALS = credentials('teckvisuals-docker')
                    withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDENTIALS, usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_PASSWORD')]) {
                        sh "docker login -u $DOCKERHUB_USERNAME -p $DOCKERHUB_PASSWORD"
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    def IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
                    def IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                script {
                    def JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')
                    def IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
                    sh "curl -v -k --user Ajoke:$JENKINS_API_TOKEN -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=$IMAGE_TAG' '$JENKINS_SERVER_URL/job/teckvisuals-CD-job/buildWithParameters?token=gitops-token'"
                }
            }
        }
    }

    post {
        failure {
            emailext subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed",
            body: 'The build has failed. Check the Jenkins console for more details.',
            to: 'ajokecloud@gmail.com'
        }
        success {
            emailext subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful",
            body: "The build was successful. You can access the artifacts at $JENKINS_SERVER_URL/job/\${env.JOB_NAME}/\${env.BUILD_NUMBER}/artifact/",
            to: 'ajokecloud@gmail.com'
        }
    }
}
