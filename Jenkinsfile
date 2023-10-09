pipeline {
    agent { label 'Agent-1' }
    tools {
        maven 'Maven3'
        //jdk "java-17-openjdk"
    }
    
    environment {
        APP_NAME = 'teckvisuals-sample-project'
        RELEASE = '1.0.0'
        DOCKER_USER = 'teckvisuals'
        DOCKER_PASS = 'teckvisuals-docker'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        GITHUB_URL = 'https://github.com/teckvisuals/register-app.git'
        JENKINS_SERVER_URL = 'http://3.96.131.176:8080'
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
                sh 'mvn clean package' 
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
                    withSonarQubeEnv(credentialsId: 'Sonar-Jenks-Cred') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        //stage('Quality Gate') {
           // steps {
            //    timeout(time: 5, unit: 'SECONDS') {
             //       waitForQualityGate abortPipeline: true
           //     }
         //   }
       // }

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
            to: 'teckvisual1@gmail.com'
        }
        success {
            emailext subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful",
            body: "The build was successful. You can access the artifacts at $JENKINS_SERVER_URL/job/\${env.JOB_NAME}/\${env.BUILD_NUMBER}/artifact/",
            to: 'teckvisual1@gmail.com'
        }
    }
}
