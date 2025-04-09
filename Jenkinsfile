pipeline {
    agent any

    environment {
        DEPLOY_DIR = '/var/www/react-app'
        SSH_SERVER = '18.233.101.14'
        SSH_USER = 'ubuntu'
        CI = 'false'
        SSH_KEY = credentials('web-hook') // Make sure this credential exists
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
                git(
                    url: 'https://github.com/karthick004/reactapp.git',
                    branch: 'main',
                    credentialsId: 'git-hub',
                    poll: false,
                    changelog: false
                )
                
                sh '''
                    echo "Verifying project structure..."
                    [ -d client ] || { echo "Error: Missing client directory"; exit 1; }
                    [ -d server ] || { echo "Error: Missing server directory"; exit 1; }
                    [ -f client/public/index.html ] || { echo "Error: Missing client/public/index.html"; exit 1; }
                    [ -f client/src/index.js ] || { echo "Error: Missing client/src/index.js"; exit 1; }
                    [ -f client/package.json ] || { echo "Error: Missing client/package.json"; exit 1; }
                '''
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('Client') {
                    steps {
                        dir('client') {
                            sh '''
                                npm install --legacy-peer-deps
                                npm install ajv@^8.0.0 ajv-keywords@^5.0.0 --save-exact
                            '''
                        }
                    }
                }
                stage('Server') {
                    steps {
                        dir('server') {
                            sh '''
                                if [ -f package.json ]; then
                                    npm install --legacy-peer-deps
                                else
                                    echo "No server package.json found - skipping"
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
                        NODE_OPTIONS="--max-old-space-size=4096" npm run build
                        [ -d build ] || { echo "Error: Build failed"; exit 1; }
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    // Write SSH key to temporary file
                    writeFile file: 'ssh_key', text: "${SSH_KEY}"
                    sh 'chmod 600 ssh_key'
                    
                    try {
                        // Deploy client build
                        sh """
                            rsync -avz --delete --progress -e 'ssh -i ssh_key -o StrictHostKeyChecking=no' \
                                client/build/ ${SSH_USER}@${SSH_SERVER}:${DEPLOY_DIR}/
                        """
                        
                        // Restart PM2 process
                        sh """
                            ssh -i ssh_key -o StrictHostKeyChecking=no ${SSH_USER}@${SSH_SERVER} "
                                if pm2 id react-app > /dev/null 2>&1; then
                                    pm2 restart react-app
                                else
                                    pm2 serve ${DEPLOY_DIR} 3000 --name 'react-app' --spa
                                fi
                            "
                        """
                        
                        // Optional: Deploy server code if needed
                        if (fileExists('server/package.json')) {
                            sh """
                                rsync -avz --delete --progress -e 'ssh -i ssh_key -o StrictHostKeyChecking=no' \
                                    --exclude='node_modules' \
                                    server/ ${SSH_USER}@${SSH_SERVER}:/opt/react-app-server/
                                
                                ssh -i ssh_key -o StrictHostKeyChecking=no ${SSH_USER}@${SSH_SERVER} "
                                    cd /opt/react-app-server && \
                                    npm install --production && \
                                    pm2 restart server-process || pm2 start server.js --name 'server-process'
                                "
                            """
                        }
                    } finally {
                        // Clean up SSH key
                        sh 'rm -f ssh_key'
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
            echo "Deployment completed successfully"
        }
        failure {
            echo "Deployment failed - check logs for details"
        }
    }
}
