pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('del-docker-hub-auth')
    }

    stages {

        stage('checkout') {
            steps {
                // Checkout the code from the Git repository
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
        stage('File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
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

        stage('Show Public IP') {
            steps {
                // Fetch and display the public IP address
                sh 'curl ifconfig.io'
            }
        }
       stage('Docker Image Scan') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.html devopseasylearning/s5wesley-do-it-yourself-ui:${BUILD_NUMBER}'
            }
        } 

        stage('Push ui-image') {
            when {
                expression {
                    env.GIT_BRANCH == 'origin/main'
                }
            }
            steps {
                sh '''
                    id
                    docker push devopseasylearning/s5wesley-do-it-yourself-ui:${BUILD_NUMBER}
                '''
            }
        }

        // Optional deployment stage
        // stage('Trigger Deployment') {
        //     agent { 
        //         label 'deploy' 
        //     }
        //     when {
        //         branch 'main'
        //     }
        //     steps {
        //         sh '''
        //             TAG=${BUILD_NUMBER}
        //             rm -rf s5wesley-do-it-yourself-automation || true
        //             git clone git@github.com:DEL-ORG/s5wesley-do-it-yourself-automation.git 
        //             cd s5wesley-do-it-yourself-automation/chart
        //             yq eval '.ui.tag = "'"$TAG"'"' -i dev-values.yaml
        //             git config --global user.name "s5wesley"
        //             git config --global user.email wesleymbarga@gmail.com 
        //             git status
        //             git add -A
        //             git commit -m "Updating ui tag to $TAG"
        //             git push origin main
        //         '''
        //     }
        // }

    }

    post {
        always {
            deleteDir() // Cleanup the workspace after the build completes
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
