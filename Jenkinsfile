pipeline {
    agent any

    environment {
        DEPLOY_DIR = '/var/www/react-app'
        SSH_SERVER = 'your-server-ip'
        SSH_USER = 'ubuntu'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build Production') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Deploy to Server') {
            steps {
                sshagent(['your-ssh-credentials-id']) {
                    sh """
                        rsync -avz --delete -e ssh build/ ${SSH_USER}@${SSH_SERVER}:${DEPLOY_DIR}/
                        ssh ${SSH_USER}@${SSH_SERVER} "
                            cd ${DEPLOY_DIR} && 
                            pm2 restart react-app || pm2 serve build 3000 --name 'react-app' --spa
                        "
                    """
                }
            }
        }
    }

    post {
        success {
            slackSend(color: 'good', message: "✅ Deployment Successful: ${env.JOB_NAME} ${env.BUILD_NUMBER}")
        }
        failure {
            slackSend(color: 'danger', message: "❌ Deployment Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}")
        }
    }
}
