pipeline {
    agent any

    environment {
        DEPLOY_DIR = '/var/www/app-cloudmasa/client'
        SSH_SERVER = '54.89.22.70'
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
                sh 'git config --global filter.lfs.smudge "git-lfs smudge --skip"'
                sh 'git config --global filter.lfs.required false'
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'master']],
                    extensions: [[
                        $class: 'CloneOption',
                        depth: 1,
                        shallow: true
                    ]],
                    userRemoteConfigs: [[
                        credentialsId: 'sandy-token',
                        url: 'https://github.com/CloudMasa-Tech/app-cloudmasa.git'
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

                        [ -d dist ] || { echo "Error: Build failed - no dist directory created"; exit 1; }
                        [ -f dist/index.html ] || { echo "Error: Missing dist/index.html"; exit 1; }
                        ls dist/assets/index-*.js >/dev/null 2>&1 || { echo "Error: Missing main JS bundle in dist/assets"; exit 1; }
                    '''
                }
            }
        }

        stage('Prepare Deployment') {
            steps {
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

        stage('Deploy') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'web-hook',
                        keyFileVariable: 'SSH_KEY_FILE',
                        usernameVariable: 'SSH_USERNAME'
                    )]) {
                        sh '''
                            echo "Deploying application to Apache2 server at $SSH_SERVER..."

                            # Sync build to temp location
                            rsync -avz --delete --progress \
                                -e "ssh -i $SSH_KEY_FILE -o StrictHostKeyChecking=no" \
                                client/dist/ "$SSH_USER@$SSH_SERVER:/tmp/react-build/"

                            # Move to deployment directory on remote host
                            ssh -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no "$SSH_USER@$SSH_SERVER" '
                                set -e
                                DEPLOY_DIR="/var/www/app-cloudmasa/client"

                                echo "Preparing Apache deployment directory at $DEPLOY_DIR..."

                                mkdir -p "$DEPLOY_DIR"
                                rm -rf "$DEPLOY_DIR"/*
                                cp -r /tmp/react-build/* "$DEPLOY_DIR"
                                chown -R www-data:www-data "$DEPLOY_DIR"

                                echo "Restarting Apache2 service..."
                                systemctl restart apache2

                                echo "Deployment complete. Listing deployed files:"
                                ls -lah "$DEPLOY_DIR"
                            '
                        '''
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
            echo "✅ Deployment completed successfully"
        }
        failure {
            echo "❌ Deployment failed - check logs for details"
        }
    }
}
