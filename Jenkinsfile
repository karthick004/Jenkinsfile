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
                
                // Verify critical files exist
                sh '''
                    echo "Verifying required files..."
                    [ -f public/index.html ] || { echo "Error: Missing public/index.html"; exit 1; }
                    [ -f src/index.js ] || { echo "Error: Missing src/index.js"; exit 1; }
                    [ -f package.json ] || { echo "Error: Missing package.json"; exit 1; }
                    
                    # Create missing reportWebVitals.js if needed
                    if [ ! -f src/reportWebVitals.js ]; then
                        echo "Creating default reportWebVitals.js..."
                        cat > src/reportWebVitals.js << 'EOL'
                        const reportWebVitals = onPerfEntry => {
                          if (onPerfEntry && onPerfEntry instanceof Function) {
                            import('web-vitals').then(({ getCLS, getFID, getFCP, getLCP, getTTFB }) => {
                              getCLS(onPerfEntry);
                              getFID(onPerfEntry);
                              getFCP(onPerfEntry);
                              getLCP(onPerfEntry);
                              getTTFB(onPerfEntry);
                            });
                          }
                        };
                        
                        export default reportWebVitals;
                        EOL
                    fi
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    # Force Node.js version check
                    echo "Using Node.js version: $(node -v)"
                    echo "Using npm version: $(npm -v)"
                    
                    # Clean install with legacy peer deps
                    npm install --legacy-peer-deps
                    
                    # Fix potential module issues
                    npm install ajv@^8.0.0 ajv-keywords@^5.0.0 --save-exact
                '''
            }
        }

        stage('Build Production') {
            steps {
                sh '''
                    # Verify node_modules exists
                    [ -d node_modules ] || { echo "Error: node_modules missing"; exit 1; }
                    
                    # Run build with increased memory
                    NODE_OPTIONS="--max-old-space-size=4096" npm run build
                    
                    # Verify build output
                    [ -d build ] || { echo "Error: Build failed - no build directory"; exit 1; }
                    [ -f build/index.html ] || { echo "Error: Missing build/index.html"; exit 1; }
                '''
            }
        }

        stage('Deploy to Server') {
            steps {
                sshagent(['web-hook']) {
                    script {
                        try {
                            sh """
                                rsync -avz --delete --progress -e ssh build/ ${SSH_USER}@${SSH_SERVER}:${DEPLOY_DIR}/
                            """
                            
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
