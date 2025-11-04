pipeline {
    agent any

    environment {
        SONAR_URL = 'http://<your-sonarqube-public-ip>:9000'
        SONAR_TOKEN = credentials('sonar-token')
        APP_IP = 'http://<your-jenkins-public-ip>'
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "📥 Checking out source code..."
                git branch: 'main', url: 'https://github.com/tanmayrannavare/devsecops.git'
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
                    sh '''
                        sonar-scanner \
                            -Dsonar.projectKey=webapp \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=$SONAR_URL \
                            -Dsonar.login=$SONAR_TOKEN
                    '''
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
                sh '''
                    docker run --rm --add-host=host.docker.internal:host-gateway \
                        -v $(pwd):/zap/wrk/ -t ghcr.io/zaproxy/zaproxy \
                        zap-baseline.py -t $APP_IP -r zap-report.html || true
                '''
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
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
