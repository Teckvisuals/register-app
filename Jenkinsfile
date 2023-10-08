pipeline {
    agent any

    environment {
        APP_NAME = 'teckvisuals-sample-project'
        RELEASE = '1.0.0'
        DOCKER_USER = 'teckvisuals'
        GITHUB_URL = 'https://github.com/teckvisuals/register-app.git'
        JENKINS_SERVER_URL = 'http://172.31.0.72:8080'
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
    }
