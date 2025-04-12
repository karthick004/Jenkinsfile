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
                        NODE_OPTIONS="--max-old-space-size=4096" npm run build_
