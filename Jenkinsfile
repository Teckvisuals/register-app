pipeline {
    agent { label 'Agent-1' }
    tools {
        maven 'Maven3'
        //jdk 'Java17'
    }
    
    environment {
        APP_NAME = 'register-app-pipeline'
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

        stage('Quality Gate') {
            steps {
                script {
                    sleep(10)
                    def qualitygate = waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-Jenks-Cred'
                    if (qualitygate.status != "OK") {
                        error("Quality Gate failed: ${qualitygate.status}")
                    }
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
                 docker.withRegistry('', DOCKER_PASS) {
                     def IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
                     def IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
                     def docker_image = docker.image("${IMAGE_NAME}:${IMAGE_TAG}")
                     docker_image.push()
                     docker_image.push('latest')
                 }
              }
          }
      }
        
     stage("Build & Push Docker Image") {
         steps {
             script {
                 docker.withRegistry('',DOCKER_PASS) {
                     docker_image = docker.build "${IMAGE_NAME}"
                }

                 docker.withRegistry('',DOCKER_PASS) {
                     docker_image.push("${IMAGE_TAG}")
                     docker_image.push('latest')
                 }
             }
         }
     }


    stage("Trivy Scan") {
        steps {
            script {
	            sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image teckvisuals/register-app-pipeline:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table')
            }
        }
    }

    stage ('Cleanup Artifacts') {
        steps {
            script {
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                sh "docker rmi ${IMAGE_NAME}:latest"
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
