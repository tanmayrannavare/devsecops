pipeline {
    agent any

    environment {
        SONAR_URL = 'http://15.207.246.187:9000'  // replace with your SonarQube EC2 public IP
        SONAR_TOKEN = credentials('sonar-token')              // Jenkins credential ID for Sonar token
        APP_IP = 'http://http://13.126.142.41'         // replace with your app’s running IP
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "📥 Checking out source code from main branch..."
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
                echo "🔍 Running static code analysis (SonarQube)..."
                withSonarQubeEnv('SonarQube') {
                    withEnv(["PATH+SONAR=${tool 'SonarScanner'}/bin"]) {
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
        }

        stage('SCA - Trivy Image Scan') {
            steps {
                echo "🧰 Running Trivy container scan..."
                sh '''
                    trivy image --exit-code 0 --severity HIGH,CRITICAL webapp:latest || true
                '''
            }
        }

        stage('DAST - OWASP ZAP Scan') {
            steps {
                echo "🧪 Running OWASP ZAP DAST scan..."
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
                echo "🚀 Deploying containerized web app..."
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
            echo "✅ DevSecOps Pipeline executed successfully!"
        }
        failure {
            echo "❌ Pipeline failed! Check logs in Jenkins console."
        }
    }
}
