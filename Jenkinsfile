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
                    extensions: [[
                        $class: 'CloneOption',
                        depth: 1,
                        shallow: true
                    ]],
                    userRemoteConfigs: [[
                        credentialsId: 'git-hub',
                        url: 'https://github.com/CloudMasa-Tech/app-cloudmasa.git/'
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
                                npm install @babel/plugin-transform-private-property-in-object --save-dev
                                npm audit fix || true
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
                                    npm ci --legacy-peer-deps --omit=dev
                                    npm audit fix || true
                                else
                                    echo "No server package.json found - skipping server dependencies"
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
                        
                        # Verify build output
                        [ -f build/index.html ] || { echo "Error: Missing build/index.html"; exit 1; }
                        [ -f build/static/js/main*.js ] || { echo "Error: Missing main JS bundle"; exit 1; }
                    '''
                }
            }
        }

        stage('Prepare Deployment') {
            steps {
                script {
                    sh '''
                        echo "Checking for rsync..."
                        if ! command -v rsync >/dev/null; then
                            echo "Installing rsync..."
                            if command -v apt-get >/dev/null; then
                                apt-get update && apt-get install -y rsync
                            elif command -v yum >/dev/null; then
                                yum install -y rsync
                            else
                                echo "Error: Could not determine package manager to install rsync"
                                exit 1
                            fi
                        fi
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
                        sh """
                            echo "Deploying application to ${SSH_SERVER}..."
                            
                            ssh -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no ${SSH_USER}@${SSH_SERVER} \
                                "mkdir -p ${DEPLOY_DIR}"
                            
                            rsync -avz --delete --progress \
                                -e "ssh -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no" \
                                client/build/ ${SSH_USER}@${SSH_SERVER}:${DEPLOY_DIR}/
                            
                            DEPLOYED_FILES=\$(ssh -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no ${SSH_USER}@${SSH_SERVER} \
                                "ls -1 ${DEPLOY_DIR} | wc -l")
                            if [ "\$DEPLOYED_FILES" -lt 5 ]; then
                                echo "Error: Deployment verification failed - too few files deployed"
                                exit 1
                            fi
                            
                            ssh -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no ${SSH_USER}@${SSH_SERVER} '
                                if ! command -v pm2 >/dev/null; then
                                    echo "Installing PM2..."
                                    npm install -g pm2
                                fi
                                
                                pm2 delete react-app 2>/dev/null || true
                                
                                pm2 serve ${DEPLOY_DIR} 3000 --name "react-app" --spa
                                
                                pm2 save
                                pm2 startup 2>/dev/null || true
                                
                                echo "PM2 process list:"
                                pm2 list
                            '
                        """
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
