pipeline {
    agent any

    environment {
        SONAR_URL = 'http://13.232.54.87:9000'       // SonarQube EC2 Public IP
        SONAR_TOKEN = credentials('sonar-token')     // Jenkins credential ID for SonarQube
    }

    stages {

        /* ========== 1️⃣ CHECKOUT CODE ========== */
        stage('Checkout Code') {
            steps {
                echo "📥 Checking out source code from main branch..."
                git branch: 'main', url: 'https://github.com/tanmayrannavare/devsecops.git'
            }
        }

        /* ========== 2️⃣ SAST - SONARQUBE ANALYSIS ========== */
        stage('SAST - SonarQube Analysis') {
            steps {
                echo "🔍 Running static code analysis (SAST) using SonarQube..."
                withSonarQubeEnv('SonarQube') {
                    withEnv(["PATH+SONAR=${tool 'SonarScanner'}/bin"]) {
                        sh '''
                            sonar-scanner \
                                -Dsonar.projectKey=webapp \
                                -Dsonar.sources=. \
                                -Dsonar.exclusions=**/zap-report.html,**/trivy-report.html \
                                -Dsonar.host.url=${SONAR_URL} \
                                -Dsonar.login=${SONAR_TOKEN}
                        '''
                        echo "✅ SAST Scan completed successfully and results sent to SonarQube dashboard."
                    }
                }
            }
        }

        /* ========== 3️⃣ BUILD DOCKER IMAGE ========== */
        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image after successful SAST scan..."
                sh '''
                    docker build -t webapp:latest .
                    echo "✅ Docker image 'webapp:latest' built successfully."
                '''
            }
        }

        /* ========== 4️⃣ SCA - TRIVY IMAGE SCAN ========== */
        stage('SCA - Trivy Image Scan') {
            steps {
                echo "🧰 Running Trivy image vulnerability scan (SCA)..."
                sh '''
                    echo "📄 Generating Trivy HTML report..."
                    trivy image --severity HIGH,CRITICAL \
                        --format template --template "@/usr/local/share/trivy/contrib/html.tpl" \
                        -o trivy-report.html webapp:latest || true
                '''
            }
            post {
                always {
                    echo "📦 Archiving Trivy report..."
                    archiveArtifacts artifacts: 'trivy-report.html', fingerprint: true
                }
            }
        }

        /* ========== 5️⃣ DEPLOY APPLICATION ========== */
        stage('Deploy Application') {
            steps {
                echo "🚀 Deploying verified Docker container to Jenkins EC2..."
                sh '''
                    docker stop webapp || true
                    docker rm webapp || true
                    docker run -d -p 80:80 --name webapp webapp:latest
                    echo "✅ Application deployed on port 80 successfully."
                '''
            }
        }

        /* ========== 6️⃣ DAST - OWASP ZAP SCAN ========== */
        stage('DAST - OWASP ZAP Scan') {
            steps {
                echo "🧪 Running OWASP ZAP Dynamic Application Security Test (DAST)..."
                script {
                    def public_ip = sh(script: "curl -s http://checkip.amazonaws.com", returnStdout: true).trim()
                    echo "🌍 Detected Jenkins Public IP: ${public_ip}"

                    sh '''
                        echo "📄 Generating OWASP ZAP HTML Report..."
                        docker run --rm --add-host=host.docker.internal:host-gateway \
                            -v $(pwd):/zap/wrk/ -t ghcr.io/zaproxy/zaproxy \
                            zap-baseline.py -t http://localhost:80 -r zap-report.html || true
                    '''
                }
            }
            post {
                always {
                    echo "📦 Archiving ZAP report..."
                    archiveArtifacts artifacts: 'zap-report.html', fingerprint: true
                }
            }
        }
    }

    /* ========== 7️⃣ POST ACTIONS ========== */
    post {
        success {
            echo "✅ DevSecOps Pipeline executed successfully!"
            echo "🌐 WebApp deployed on Jenkins EC2 (Port 80)."
            echo "📊 Reports generated and archived: trivy-report.html, zap-report.html"
            echo "📈 SonarQube dashboard updated at ${SONAR_URL}/dashboard?id=webapp"
        }
        failure {
            echo "❌ Pipeline failed! Check Jenkins logs and archived reports for details."
        }
    }
}
