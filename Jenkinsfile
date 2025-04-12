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
                        npm ci --legacy-peer-deps
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

                        [ -d dist ] || { echo "❌ Build failed - dist folder missing"; exit 1; }
                        [ -f dist/index.html ] || { echo "❌ Missing dist/index.html"; exit 1; }
                        echo "✅ Build completed successfully"
                    '''
                }
            }
        }

        stage('Prepare Deployment Tools') {
            steps {
                sh '''
                    echo "🔍 Checking required tools..."
                    command -v rsync >/dev/null 2>&1 || { 
                        echo "❌ rsync not found on Jenkins node"; 
                        exit 1; 
                    }
                    echo "✅ All tools available"
                '''
            }
        }

        stage('Configure Remote Server') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'web-hook',
                        keyFileVariable: 'SSH_KEY_FILE',
                        usernameVariable: 'SSH_USERNAME'
                    )]) {
                        sh """
                            echo "🔐 Configuring remote server..."
                            
                            # Create setup script
                            cat > remote_setup.sh << 'REMOTE_EOF'
                            #!/bin/bash
                            set -e
                            
                            echo "🔧 Updating packages..."
                            sudo apt-get update -y
                            
                            echo "🔍 Checking Apache2..."
                            if ! command -v apache2 >/dev/null 2>&1; then
                                echo "📦 Installing Apache2..."
                                sudo apt-get install -y apache2
                            fi
                            
                            echo "📂 Creating deployment directory..."
                            sudo mkdir -p ${DEPLOY_DIR}
                            sudo chown -R \$USER:\$USER ${DEPLOY_DIR}
                            sudo chmod -R 755 ${DEPLOY_DIR}
                            
                            echo "📝 Configuring Apache..."
                            if [ ! -f /etc/apache2/sites-available/app-cloudmasa.conf ]; then
                                cat << 'APACHE_EOF' | sudo tee /etc/apache2/sites-available/app-cloudmasa.conf >/dev/null
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
                            APACHE_EOF
                                
                                sudo a2dissite 000-default.conf
                                sudo a2ensite app-cloudmasa.conf
                                sudo a2enmod rewrite
                            fi
                            
                            echo "🔄 Restarting Apache..."
                            sudo systemctl restart apache2
                            REMOTE_EOF
                            
                            # Transfer and execute setup
                            scp -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no remote_setup.sh "$SSH_USER@$SSH_SERVER:~/"
                            ssh -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no "$SSH_USER@$SSH_SERVER" "chmod +x ~/remote_setup.sh && ~/remote_setup.sh"
                        """
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
                        sh """
                            echo "🚀 Deploying to ${DEPLOY_DIR}..."
                            
                            # Sync build files
                            rsync -avz --delete --progress \
                                -e "ssh -i $SSH_KEY_FILE -o StrictHostKeyChecking=no" \
                                client/dist/ "$SSH_USER@$SSH_SERVER:/tmp/react-build/"
                            
                            # Final deployment steps
                            ssh -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no "$SSH_USER@$SSH_SERVER" << 'DEPLOY_EOF'
                                set -e
                                echo "📂 Moving files to ${DEPLOY_DIR}..."
                                sudo rm -rf ${DEPLOY_DIR}/*
                                sudo mv /tmp/react-build/* ${DEPLOY_DIR}/
                                
                                echo "🔒 Setting permissions..."
                                sudo chown -R www-data:www-data ${DEPLOY_DIR}
                                sudo chmod -R 755 ${DEPLOY_DIR}
                                
                                echo "📝 Configuring .htaccess for React Router..."
                                cat << 'HTACCESS_EOF' | sudo tee ${DEPLOY_DIR}/.htaccess >/dev/null
                            Options -MultiViews
                            RewriteEngine On
                            RewriteBase /
                            RewriteRule ^index\\.html$ - [L]
                            RewriteCond %{REQUEST_FILENAME} !-f
                            RewriteCond %{REQUEST_FILENAME} !-d
                            RewriteRule . /index.html [L]
                            HTACCESS_EOF
                                
                                echo "🔄 Final Apache restart..."
                                sudo systemctl restart apache2
                                
                                echo "✅ Deployment complete!"
                                echo "📋 Directory listing:"
                                ls -la ${DEPLOY_DIR}
                            DEPLOY_EOF
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
        }
    }
}
