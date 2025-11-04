pipeline {
    agent any

    environment {
        SONAR_URL = 'http://localhost:9000'
        SONAR_TOKEN = credentials('sonar-token')   // Jenkins credential ID
        SLACK_CHANNEL = '#devsecops-alerts'       // your Slack channel
        APP_IP = 'http://192.168.56.120'          // your live app IP
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "📥 Checking out source code..."
                git 'https://github.com/<your-username>/<your-repo-name>.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image..."
                sh 'docker build -t webapp:latest .'
            }
        }

        stage('SAST - SonarQube Analysis') {
            steps {
                echo "🔍 Running static analysis..."
                withSonarQubeEnv('SonarQube') {
                    sh """
                    sonar-scanner \
                        -Dsonar.projectKey=webapp \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=$SONAR_URL \
                        -Dsonar.login=$SONAR_TOKEN
                    """
                }
            }
        }

        stage('SCA - Trivy Image Scan') {
            steps {
                echo "🧰 Scanning image for vulnerabilities..."
                sh 'trivy image --exit-code 0 --severity HIGH,CRITICAL webapp:latest || true'
            }
        }

        stage('DAST - OWASP ZAP Scan') {
            steps {
                echo "🧪 Running OWASP ZAP security test..."
                sh """
                docker run --rm --add-host=host.docker.internal:host-gateway \
                    -v $(pwd):/zap/wrk/ -t ghcr.io/zaproxy/zaproxy \
                    zap-baseline.py -t $APP_IP -r zap-report.html || true
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap-report.html', fingerprint: true
                }
            }
        }

        stage('Deploy Application') {
            steps {
                echo "🚀 Deploying application container..."
                sh '''
                docker stop webapp || true
                docker rm webapp || true
                docker run -d -p 80:80 --name webapp webapp:latest
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline succeeded!"
            slackSend(channel: "${SLACK_CHANNEL}", message: "✅ DevSecOps Pipeline passed successfully for webapp.")
        }
        failure {
            echo "❌ Pipeline failed!"
            slackSend(channel: "${SLACK_CHANNEL}", message: "❌ DevSecOps Pipeline failed for webapp. Check Jenkins logs.")
        }
    }
}
