pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('del-docker-hub-auth')
    }

    stages {

        stage('unit test') {
            agent {
                docker { 
                    image 'maven:3.8.5-openjdk-18' 
                    args '-u root' 
                }

            }
            steps {
                sh '''
                cd ui
                mvn clean test
                ls
                '''
            }
        }

        stage('SonarQube analysis') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:5.0.1'
                }
            }
            environment {
                CI = 'true'
                scannerHome = '/opt/sonar-scanner'
            }
            steps {
                withSonarQubeEnv('Sonar') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'del-docker-hub-auth', usernameVariable: 'DOCKERHUB_CREDENTIALS_USR', passwordVariable: 'DOCKERHUB_CREDENTIALS_PSW')]) {
                    // Docker login steps would go here
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    cd ${WORKSPACE}/ui
                    docker build -t devopseasylearning/s7rosine-ui:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Push ui-image') {
            when {
                expression { env.GIT_BRANCH == 'origin/main' }
            }
            steps {
                sh '''
                    id
                    docker push devopseasylearning/s7rosine-ui:${BUILD_NUMBER}
                '''
            }
        }

        stage('Trigger Deployment') {
            agent { 
                label 'deploy' 
            }
            when {
                expression { env.GIT_BRANCH == 'origin/main' }
            }
            steps {
                sh '''
                    TAG=${BUILD_NUMBER}
                    rm -rf s7rosine-automation || true
                    git clone git@github.com:DEL-ORG/s7rosine-automation.git 
                    cd s7rosine-automation/chart
                    yq eval '.ui.tag = "'"$TAG"'"' -i dev-values.yaml
                    git config --global user.name "rosinebel"
                    git config --global user.email rosinemuku@yahoo.com 
                    git status
                    git add -A
                    git commit -m "Updating ui tag to $TAG"
                    git push origin main
                '''
            }
        }
    }

    post {
        success {
            slackSend (
                channel: '#development-alerts',
                color: 'good',
                message: "SUCCESSFUL: Application s7rosine-ui Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
            )
        }

        unstable {
            slackSend (
                channel: '#development-alerts',
                color: 'warning',
                message: "UNSTABLE: Application s7rosine-ui Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
            )
        }

        failure {
            slackSend (
                channel: '#development-alerts',
                color: '#FF0000',
                message: "FAILURE: Application s7rosine-ui Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
            )
        }

        cleanup {
            deleteDir()
        }
    }
}
