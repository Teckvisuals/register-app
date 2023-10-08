pipeline {
    agent { label 'Agent-1' }
    tools {
        jdk 'Java17'
        maven 'Maven3'
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
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'Teckvisuals-Git-Cred', url: GITHUB_URL
            }
        }

        stage("Build Application") {
            steps {
                sh 'mvn clean package'
            }
        }

        stage("Test Application") {
            steps {
                sh 'mvn test'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'Sonar-Jenks-Cred') { 
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-Jenks-Cred'
                }
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_USER) {
                        docker_image = docker.build "${IMAGE_NAME}:${IMAGE_TAG}"
                        docker_image.push()
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh "docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table"
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh "curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-13-232-128-192.ap-south-1.compute.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'"
                }
            }
        }
    }

    post {
        failure {
            emailext subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed",
            body: 'The build has failed. Check the Jenkins console for more details.',
            to:'teckvisual1@gmail.com'
        }
        success {
            emailext subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful",
            body: "The build was successful. You can access the artifacts at $JENKINS_SERVER_URL/job/\${env.JOB_NAME}/\${env.BUILD_NUMBER}/artifact/",
            to:'teckvisual1@gmail.com'
        }
    }
}
