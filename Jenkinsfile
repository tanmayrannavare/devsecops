pipeline {
    agent any

    environment {
        SONAR_URL = 'http://13.232.54.87:9000'
        SONAR_TOKEN = credentials('sonar-token')
        APP_PORT = '80'
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

        stage('Deploy Application') {
            steps {
                echo "🚀 Deploying containerized web app..."
                sh '''
                    docker stop webapp || true
                    docker rm webapp || true
                    docker run -d -p 80:80 --name webapp webapp:latest
                '''
            }
        }

        stage('DAST - OWASP ZAP Scan') {
            steps {
                echo "🧪 Running OWASP ZAP DAST scan..."
                script {
                    // Dynamically fetch the public IP of the Jenkins EC2 instance
                    def public_ip = sh(script: "curl -s http://checkip.amazonaws.com", returnStdout: true).trim()
                    echo "🌍 Detected Jenkins Public IP: ${public_ip}"

                    sh """
                        docker run --rm --add-host=host.docker.internal:host-gateway \
                            -v $(pwd):/zap/wrk/ -t ghcr.io/zaproxy/zaproxy \
                            zap-baseline.py -t http://${public_ip}:${APP_PORT} -r zap-report.html || true
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap-report.html', fingerprint: true
                }
            }
        }
    }

    post {
        success {
            echo "✅ DevSecOps Pipeline executed successfully!"
        }
        failure {
            echo "❌ Pipeline failed! Check logs in Jenkins console."
        }
    }
}
