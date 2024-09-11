pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('del-docker-hub-auth')
    }

    stages {
        stage('checkout') {
            steps {
                git branch: 'main', 
                    credentialsId: 'github-ssh', 
                    url: 'git@github.com:DEL-ORG/s5wesley-do-it-yourself-ui.git'
            }
        }

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

        stage('File System Scan') {
            agent {
                docker { 
                    image 'bitnami/trivy:latest' 
                    args '--entrypoint="" -u root'
                }
            } 

            steps {
                sh ''' 
                   cd ui
                   trivy fs --format table -o trivy-fs-report.html .
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
                    sh 'docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    cd ${WORKSPACE}/ui
                    docker build -t devopseasylearning/s5wesley-do-it-yourself-ui:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Push ui-image') {
            steps {
                sh '''
                    docker push devopseasylearning/s5wesley-do-it-yourself-ui:${BUILD_NUMBER}
                '''
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.html devopseasylearning/s5wesley-do-it-yourself-ui:${BUILD_NUMBER}'
            }
        }

        stage('Show Public IP') {
            steps {
                sh 'curl ifconfig.io'
            }
        }
    }

    post {
        always {
            deleteDir()
        }

        success {
            slackSend (
                channel: '#development-alerts',
                color: 'good',
                message: "SUCCESSFUL: Application s5wesley-do-it-yourself-ui Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
            )
        }

        unstable {
            slackSend (
                channel: '#development-alerts',
                color: 'warning',
                message: "UNSTABLE: Application s5wesley-do-it-yourself-ui Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
            )
        }

        failure {
            slackSend (
                channel: '#development-alerts',
                color: '#FF0000',
                message: "FAILURE: Application s5wesley-do-it-yourself-ui Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
            )
        }
    }
}
