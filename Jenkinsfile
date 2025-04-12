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
                    ls -la
                    [ -d client ] || { echo "❌ Missing 'client' directory"; exit 1; }
                    [ -f client/package.json ] || { echo "❌ Missing client/package.json"; exit 1; }
                    echo "✅ Project structure verified"
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('client') {
                    sh '''
                        echo "📦 Installing client dependencies..."
                        npm install || npm ci --legacy-peer-deps
                        npm install @babel/plugin-transform-private-property-in-object --save-dev
                        npm audit fix || true
                        echo "✅ Dependencies installed"
                    '''
                }
            }
        }

        stage('Build Client') {
            steps {
                dir('client') {
                    sh '''
                        echo "🔨 Building client application..."
                        NODE_OPTIONS="--max-old-space-size=4096" npm run build

                        echo "Build artifacts:"
                        ls -la dist/
                        [ -d dist ] || { echo "❌ Build failed - dist folder missing"; exit 1; }
                        [ -f dist/index.html ] || { echo "❌ Missing dist/index.html"; exit 1; }
                        echo "✅ Build completed successfully"
                    '''
                }
            }
        }

        stage('Verify Deployment Tools') {
            steps {
                sh '''
                    echo "🔍 Checking tools..."
                    echo "Node version:"
                    node --version
                    echo "npm version:"
                    npm --version
                    echo "rsync version:"
                    rsync --version || { echo "❌ rsync not found"; exit 1; }
                    echo "✅ All tools verified"
                '''
            }
        }

        stage('Remote Server Setup') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'web-hook',
                        keyFileVariable: 'SSH_KEY_FILE',
                        usernameVariable: 'SSH_USERNAME'
                    )]) {
                        // First verify SSH connection works
                        sh """
                            echo "🔐 Testing SSH connection..."
                            ssh -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no -o ConnectTimeout=10 "$SSH_USER@$SSH_SERVER" "echo 'SSH connection successful'"
                            
                            echo "🛠 Setting up remote server..."
                            ssh -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no "$SSH_USER@$SSH_SERVER" '
                                set -ex
                                echo "Updating packages..."
                                sudo apt-get update -y
                                
                                echo "Installing Apache if needed..."
                                if ! command -v apache2 >/dev/null 2>&1; then
                                    sudo apt-get install -y apache2
                                fi
                                
                                echo "Creating deployment directory: ${DEPLOY_DIR}"
                                sudo mkdir -p "${DEPLOY_DIR}"
                                sudo chown -R $SSH_USER:$SSH_USER "${DEPLOY_DIR}"
                                sudo chmod -R 755 "${DEPLOY_DIR}"
                                
                                echo "Configuring Apache..."
                                sudo a2enmod rewrite
                                cat <<EOF | sudo tee /etc/apache2/sites-available/app-cloudmasa.conf
<VirtualHost *:80>
    ServerAdmin admin@cloudmasa.com
    ServerName ${SSH_SERVER}
    DocumentRoot ${DEPLOY_DIR}
    
    <Directory ${DEPLOY_DIR}>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
        DirectoryIndex index.html
    </Directory>
    
    ErrorLog \${APACHE_LOG_DIR}/error.log
    CustomLog \${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
EOF
                                
                                sudo a2dissite 000-default.conf
                                sudo a2ensite app-cloudmasa.conf
                                sudo systemctl restart apache2
                            '
                        """
                    }
                }
            }
        }

        stage('Deploy Application') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'web-hook',
                        keyFileVariable: 'SSH_KEY_FILE',
                        usernameVariable: 'SSH_USERNAME'
                    )]) {
                        sh """
                            echo "🚀 Starting deployment..."
                            
                            echo "📦 Syncing build files..."
                            rsync -avz --delete --progress \
                                -e "ssh -i $SSH_KEY_FILE -o StrictHostKeyChecking=no" \
                                client/dist/ "$SSH_USER@$SSH_SERVER:/tmp/react-build/"
                            
                            echo "🏗 Moving files to production..."
                            ssh -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no "$SSH_USER@$SSH_SERVER" '
                                set -ex
                                echo "Cleaning deployment directory..."
                                sudo rm -rf "${DEPLOY_DIR}"/*
                                
                                echo "Moving files..."
                                sudo mv /tmp/react-build/* "${DEPLOY_DIR}/"
                                
                                echo "Setting permissions..."
                                sudo chown -R www-data:www-data "${DEPLOY_DIR}"
                                sudo chmod -R 755 "${DEPLOY_DIR}"
                                
                                echo "Configuring .htaccess..."
                                cat <<EOF | sudo tee "${DEPLOY_DIR}/.htaccess"
Options -MultiViews
RewriteEngine On
RewriteBase /
RewriteRule ^index\\.html$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.html [L]
EOF
                                
                                echo "Restarting Apache..."
                                sudo systemctl restart apache2
                                
                                echo "Verifying deployment..."
                                ls -la "${DEPLOY_DIR}"
                            '
                        """
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
            echo "✅ Pipeline completed successfully!"
            script {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'web-hook',
                    keyFileVariable: 'SSH_KEY_FILE',
                    usernameVariable: 'SSH_USERNAME'
                )]) {
                    echo "🌐 Your React app is now live at: http://${SSH_SERVER}"
                }
            }
        }
        failure {
            echo "❌ Pipeline failed. Please check the logs for errors."
            script {
                // Add any failure notifications here
            }
        }
    }
}
