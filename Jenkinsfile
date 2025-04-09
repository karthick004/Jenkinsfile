pipeline {
    agent any

    environment {
        DEPLOY_DIR = '/var/www/react-app'
        SSH_SERVER = '18.233.101.14'
        SSH_USER = 'ubuntu'
        CI = 'false'  // Prevent build failures on warnings
    }

    tools {
        nodejs 'Node18'  // Must use Node.js 16.x or 18.x (LTS versions)
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()  // Start with clean workspace
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
                
                // Verify project structure
                sh '''
                    echo "Verifying project structure..."
                    if [ ! -d client ]; then
                        echo "Error: Missing client directory"
                        exit 1
                    fi
                    if [ ! -d server ]; then
                        echo "Error: Missing server directory"
                        exit 1
                    fi
                    
                    # Verify client files
                    echo "Checking client files..."
                    if [ ! -f client/public/index.html ]; then
                        echo "Error: Missing client/public/index.html"
                        exit 1
                    fi
                    if [ ! -f client/src/index.js ]; then
                        echo "Error: Missing client/src/index.js"
                        exit 1
                    fi
                    if [ ! -f client/package.json ]; then
                        echo "Error: Missing client/package.json"
                        exit 1
                    fi
                    
                    # Verify server files (if needed)
                    if [ ! -f server/package.json ]; then
                        echo "Warning: Missing server/package.json (optional)"
                    fi
                '''
            }
        }

        stage('Install Client Dependencies') {
            steps {
                dir('client') {
                    sh '''
                        echo "Installing client dependencies..."
                        npm install --legacy-peer-deps
                        npm install ajv@^8.0.0 ajv-keywords@^5.0.0 --save-exact
                    '''
                }
            }
        }

        stage('Install Server Dependencies') {
            steps {
                dir('server') {
                    sh '''
                        echo "Installing server dependencies..."
                        if [ -f package.json ]; then
                            npm install --legacy-peer-deps
                        else
                            echo "No server package.json found - skipping"
                        fi
                    '''
                }
            }
        }

        stage('Build Client') {
            steps {
                dir('client') {
                    sh '''
                        echo "Building client..."
                        NODE_OPTIONS="--max-old-space-size=4096" npm run build
                        
                        if [ ! -d build ]; then
                            echo "Error: Client build failed - no build directory"
                            exit 1
                        fi
                    '''
                }
            }
        }

        stage('Deploy to Server') {
            steps {
                sshagent(['web-hook']) {
                    script {
                        try {
                            // Deploy client build
                            sh """
                                rsync -avz --delete --progress -e ssh client/build/ ${SSH_USER}@${SSH_SERVER}:${DEPLOY_DIR}/
                            """
                            
                            // Restart PM2 process
                            sh """
                                ssh ${SSH_USER}@${SSH_SERVER} "
                                    if pm2 id react-app > /dev/null 2>&1; then
                                        pm2 restart react-app
                                    else
                                        pm2 serve ${DEPLOY_DIR} 3000 --name 'react-app' --spa
                                    fi
                                "
                            """
                        } catch (err) {
                            error "Deployment failed: ${err.message}"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed - cleaning up"
        }
        failure {
            echo "Pipeline failed - check the logs for details"
        }
        success {
            echo "Pipeline completed successfully"
        }
    }
}
