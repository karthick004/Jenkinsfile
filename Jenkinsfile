pipeline {
    agent any

    environment {
        DEPLOY_DIR = '/var/www/react-app'
        SSH_SERVER = '18.233.101.14'
        SSH_USER = 'ubuntu'
        CI = 'false'
    }

    tools {
        nodejs 'Node18'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'git-hub',
                        url: 'https://github.com/karthick004/reactapp.git'
                    ]]
                ])
                
                sh '''
                    echo "Verifying project structure..."
                    [ -d client ] || { echo "Error: Missing client directory"; exit 1; }
                    [ -f client/package.json ] || { echo "Error: Missing client/package.json"; exit 1; }
                    if [ -d server ]; then
                        echo "Server directory found"
                    fi
                '''
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('Client') {
                    steps {
                        dir('client') {
                            sh '''
                                echo "Installing client dependencies..."
                                npm ci --legacy-peer-deps
                                npm install @babel/plugin-proposal-private-property-in-object --save-dev
                            '''
                        }
                    }
                }
                stage('Server') {
                    steps {
                        dir('server') {
                            sh '''
                                if [ -f package.json ]; then
                                    echo "Installing server dependencies..."
                                    npm ci --legacy-peer-deps --only=production
                                fi
                            '''
                        }
                    }
                }
            }
        }

        stage('Build Client') {
            steps {
                dir('client') {
                    sh '''
                        echo "Building client application..."
                        NODE_OPTIONS="--max-old-space-size=4096" npm run build
                        [ -d build ] || { echo "Error: Build failed - no build directory created"; exit 1; }
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'web-hook',
                        keyFileVariable: 'SSH_KEY_FILE',
                        usernameVariable: 'SSH_USERNAME'
                    )]) {
                        // Deploy client build
                        sh """
                            rsync -avz --delete --progress \
                                -e "ssh -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no" \
                                client/build/ ${SSH_USER}@${SSH_SERVER}:${DEPLOY_DIR}/
                        """
                        
                        // Restart PM2 process
                        sh """
                            ssh -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no ${SSH_USER}@${SSH_SERVER} '
                                if pm2 id react-app >/dev/null 2>&1; then
                                    pm2 restart react-app
                                else
                                    pm2 serve ${DEPLOY_DIR} 3000 --name "react-app" --spa
                                fi
                            '
                        """
                        
                        // Conditional server deployment
                        if (fileExists('server/package.json')) {
                            sh """
                                rsync -avz --delete --progress \
                                    -e "ssh -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no" \
                                    --exclude='node_modules' \
                                    server/ ${SSH_USER}@${SSH_SERVER}:/opt/react-app-server/
                                
                                ssh -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no ${SSH_USER}@${SSH_SERVER} '
                                    cd /opt/react-app-server && \
                                    npm ci --only=production && \
                                    (pm2 restart server-process || pm2 start server.js --name "server-process")
                                '
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up workspace"
            cleanWs()
        }
        success {
            slackSend(
                color: 'good',
                message: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                channel: '#deployments'
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                channel: '#deployments'
            )
        }
    }
}
