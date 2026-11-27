pipeline {
    agent any

    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    def info = checkout scm
                    env.BUILD_GIT_BRANCH = info.GIT_BRANCH
                    echo "Branch: ${env.BUILD_GIT_BRANCH}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                bat 'python -m venv .venv'
                bat '.venv\\Scripts\\python.exe -m pip install -r requirements.txt'

                dir('client') {
                    bat 'npm ci'
                }
            }
        }

        stage('Tests') {
            steps {
                bat '.venv\\Scripts\\python.exe manage.py test'
            }
        }

        stage('Build') {
            steps {
                dir('client') {
                    bat 'npm run build'
                }
            }
        }

        stage('Delivery') {
            when {
                environment name: 'BUILD_GIT_BRANCH', value: 'origin/main'
            }
            steps {
                bat 'git archive --format=zip --output=esports-source.zip HEAD'
                archiveArtifacts artifacts: 'esports-source.zip,client/dist/**', fingerprint: true
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully'
        }

        failure {
            echo 'Pipeline failed'
        }
    }
}