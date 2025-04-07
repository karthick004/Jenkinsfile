pipeline {
    agent any

    environment {
        DEPLOY_DIR = '/var/www/react-app'
        SSH_SERVER = '18.233.101.14'
        SSH_USER = 'ubuntu'
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/karthick004/reactapp.git/', 
                    branch: 'main'
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
                sshagent(['web-hook']) {
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
}
