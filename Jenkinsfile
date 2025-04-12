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
                    branches: [[name: '*/master']],
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
                    echo "🔍 Verifying project structure..."
                    [ -d client ] || { echo "❌ Missing 'client' directory"; exit 1; }
                    [ -f client/package.json ] || { echo "❌ Missing client/package.json"; exit 1; }
                    if [ -d server ]; then
                        echo "✅ Server directory found"
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
                                echo "📦 Installing client dependencies..."
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
                                    echo "📦 Installing server dependencies..."
                                    npm ci --legacy-peer-deps --omit=dev
                                    npm audit fix || true
                                else
                                    echo "ℹ️ No server/package.json found. Skipping..."
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
                        echo "🔨 Building client application..."
                        NODE_OPTIONS="--max-old-space-size=4096" npm run build

                        [ -d dist ] || { echo "❌ Build failed - dist folder missing"; exit 1; }
                        [ -f dist/index.html ] || { echo "❌ Missing dist/index.html"; exit 1; }
                        ls dist/assets/index-*.js >/dev/null 2>&1 || { echo "❌ Main JS bundle missing in dist/assets"; exit 1; }
                    '''
                }
            }
        }

        stage('Prepare Deployment Tools') {
            steps {
                sh '''
                    echo "🔍 Checking if 'rsync' is installed..."
                    if ! command -v rsync >/dev/null 2>&1; then
                        echo "📦 Installing rsync..."
                        if command -v apt-get >/dev/null; then
                            sudo apt-get update && sudo apt-get install -y rsync
                        elif command -v yum >/dev/null; then
                            sudo yum install -y rsync
                        else
                            echo "❌ No compatible package manager found to install rsync"
                            exit 1
                        fi
                    else
                        echo "✅ rsync already installed"
                    fi
                '''
            }
        }

        stage('Ensure Apache2 on Remote') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'web-hook',
                        keyFileVariable: 'SSH_KEY_FILE',
                        usernameVariable: 'SSH_USERNAME'
                    )]) {
                        sh '''
                            echo "🔐 Connecting to remote server to setup Apache2..."

                            ssh -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no "$SSH_USER@$SSH_SERVER" <<"EOF"
                                set -e

                                echo "🔍 Checking Apache2..."
                                if ! command -v apache2 >/dev/null 2>&1; then
                                    echo "📦 Installing Apache2..."
                                    sudo apt update
                                    sudo apt install -y apache2
                                    sudo systemctl enable apache2
                                else
                                    echo "✅ Apache2 already installed"
                                fi

                                echo "🔄 Restarting Apache2..."
                                sudo systemctl restart apache2
                                sudo systemctl status apache2 --no-pager || true
                            EOF
                        '''
                    }
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
                        sh '''
                            echo "🚀 Deploying client build to remote server..."

                            rsync -avz --delete --progress \
                                -e "ssh -i $SSH_KEY_FILE -o StrictHostKeyChecking=no" \
                                client/dist/ "$SSH_USER@$SSH_SERVER:/tmp/react-build/"

                            ssh -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no "$SSH_USER@$SSH_SERVER" <<"EOF"
                                set -e
                                DEPLOY_DIR="/var/www/app-cloudmasa/client"

                                echo "📁 Deploying to \$DEPLOY_DIR..."
                                sudo mkdir -p "\$DEPLOY_DIR"
                                sudo rm -rf "\$DEPLOY_DIR"/*
                                sudo cp -r /tmp/react-build/* "\$DEPLOY_DIR"
                                sudo chown -R www-data:www-data "\$DEPLOY_DIR"

                                echo "🔁 Restarting Apache2..."
                                sudo systemctl restart apache2

                                echo "✅ Deployment complete! Files at \$DEPLOY_DIR:"
                                ls -lah "\$DEPLOY_DIR"
                            EOF
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning workspace..."
            cleanWs()
        }
        success {
            echo "✅ Deployment completed successfully!"
        }
        failure {
            echo "❌ Deployment failed. Please check the logs for errors."
        }
    }
}
