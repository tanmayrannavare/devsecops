pipeline {
    agent any

    environment {
        SONAR_URL = 'http://13.232.54.87:9000'      // SonarQube EC2 Public IP
        SONAR_TOKEN = credentials('sonar-token')    // Jenkins credential ID for Sonar token
    }

    stages {

        /* ========== 1️⃣ CHECKOUT CODE ========== */
        stage('Checkout Code') {
            steps {
                echo "📥 Checking out source code from main branch..."
                git branch: 'main', url: 'https://github.com/tanmayrannavare/devsecops.git'
            }
        }

        /* ========== 2️⃣ BUILD DOCKER IMAGE ========== */
        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image..."
                sh 'docker build -t webapp:latest .'
            }
        }

        /* ========== 3️⃣ SAST - SONARQUBE ANALYSIS ========== */
        stage('SAST - SonarQube Analysis') {
            steps {
                echo "🔍 Running static code analysis with SonarQube..."
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
                    }
                }
            }
        }

        /* ========== 4️⃣ SCA - TRIVY IMAGE SCAN ========== */
        stage('SCA - Trivy Image Scan') {
            steps {
                echo "🧰 Running Trivy container vulnerability scan..."
                sh '''
                    echo "📄 Generating Trivy HTML Report..."
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
                echo "🚀 Deploying web app container on Jenkins EC2..."
                sh '''
                    docker stop webapp || true
                    docker rm webapp || true
                    docker run -d -p 80:80 --name webapp webapp:latest
                '''
            }
        }

        /* ========== 6️⃣ DAST - OWASP ZAP SCAN ========== */
        stage('DAST - OWASP ZAP Scan') {
            steps {
                echo "🧪 Running OWASP ZAP Dynamic Security Scan..."
                script {
                    def public_ip = sh(script: "curl -s http://checkip.amazonaws.com", returnStdout: true).trim()
                    echo "🌍 Detected Jenkins Public IP: ${public_ip}"

                    sh '''
                        echo "📄 Generating ZAP HTML Report..."
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
            echo "🌐 WebApp deployed on port 80."
            echo "📊 Reports generated: trivy-report.html and zap-report.html"
        }
        failure {
            echo "❌ Pipeline failed! Check logs and reports for details
