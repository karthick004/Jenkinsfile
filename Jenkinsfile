pipeline {
    agent any

    environment {
        DEPLOY_DIR = '/var/www/react-app'
        SSH_SERVER = '18.233.101.14'
        SSH_USER = 'ubuntu'
        CI = 'false'  // Prevent build failures on warnings
    }

    tools {
        nodejs 'Node18'  // Ensure Node.js is installed via Jenkins Global Tool Configuration
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
                    credentialsId: 'git-hub',  // Use your configured credentials
                    poll: false,
                    changelog: false
                )
                
                // Verify critical files exist
                sh '''
                    echo "Verifying required files..."
                    [ -f public/index.html ] || { echo "Error: Missing public/index.html"; exit 1; }
                    [ -f src/index.js ] || { echo "Error: Missing src/index.js"; exit 1; }
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install --legacy-peer-deps'  // Handle dependency conflicts
            }
        }

        stage('Build Production') {
            steps {
                sh 'npm run build'
                
                // Verify build output
                sh '''
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
                            
                            // Check if PM2 process exists before restarting
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
            // Cleanup workspace if needed
            // cleanWs()
        }
        failure {
            // slackSend color: 'danger', message: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            echo "Pipeline failed - check the logs for details"
        }
        success {
            // slackSend color: 'good', message: "Deployed successfully: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            echo "Pipeline completed successfully"
        }
    }
}
