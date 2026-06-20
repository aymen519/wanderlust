pipeline {
    agent any
    
    environment {
        SONAR_HOME = tool "SonarScanner"
        OWASP_HOME = tool 'OWASP'
        DOCKER_REGISTRY = 'https://index.docker.io/v1/'
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'
    }

    stages {
        stage("1-Github Code & Docker Setup") {
            steps {
                echo "📥 Clonage du repository..."
                git url: "https://github.com/aymen519/wanderlust", branch: "devops"
                
                echo "🛠️ Création automatique des fichiers Docker manquants..."
                sh '''
                    cat <<EOF > frontend/Dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

                    cat <<EOF > backend/Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3500
CMD ["node", "server.js"]
EOF

                    cat <<EOF > docker-compose.yml
services:
  frontend:
    image: aymen519/wanderlust-frontend:latest
    ports:
      - "3000:80"
    depends_on:
      - backend
  
  backend:
    image: aymen519/wanderlust-backend:latest
    ports:
      - "3500:3500"
    environment:
      - MONGODB_URI=mongodb://mongo:27017/wanderlust
    depends_on:
      - mongo
  
  mongo:
    image: mongo:6.0
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
EOF
                    echo "✅ Fichiers Docker créés avec succès !"
                '''
            }
        }

        stage("2-SonarQube Analysis") {
            steps {
                withSonarQubeEnv("Sonar") {
                    sh """
                        ${SONAR_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=wanderlust \
                        -Dsonar.projectKey=wanderlust \
                        -Dsonar.sources=. \
                        -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage('3-OWASP Scan') {
            steps {
                echo "🛡️ Scan OWASP..."
                
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    sh '''
                        if [ -f "$OWASP_HOME/bin/dependency-check.sh" ]; then
                            echo "🔍 Lancement scan OWASP..."
                            $OWASP_HOME/bin/dependency-check.sh \
                                --scan ./frontend \
                                --out . \
                                --format HTML \
                                --project wanderlust || true
                        fi
                    '''
                }
                
                echo "✅ Stage OWASP terminé"
            }
        }

        stage("4-Quality Gate") {
            steps {
                timeout(time: 2, unit: "MINUTES") {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage("5-Trivy Scan") {
            steps {
                echo "🔍 Scan Trivy..."
                sh "trivy fs --format table -o trivy-fs-report.html . || true"
            }
        }

        stage("6-Docker Build & Push") {
            steps {
                script {
                    echo "🐳 Connexion Docker Hub..."
                    docker.withRegistry(DOCKER_REGISTRY, DOCKER_CREDENTIALS_ID) {
                        echo "🔨 Build Frontend..."
                        def frontendApp = docker.build("aymen519/wanderlust-frontend:${BUILD_NUMBER}", "frontend")
                        frontendApp.push()
                        
                        echo "🔨 Build Backend..."
                        def backendApp = docker.build("aymen519/wanderlust-backend:${BUILD_NUMBER}", "backend")
                        backendApp.push()
                        
                        echo "📤 Push latest..."
                        frontendApp.push('latest')
                        backendApp.push('latest')
                        
                        echo "✅ Images poussées !"
                    }
                }
            }
        }

        stage("7-Deploy to Web Server") {
            steps {
                script {
                    echo "🚀 Déploiement via Ansible..."
                    sh '''
                        ansible-playbook /var/lib/jenkins/deploy.yml -i /var/lib/jenkins/inventory.yml
                    '''
                    echo "✅ Déployé sur https://100.57.116.52"
                }
            }
        }

        stage("8-DAST ZAP Scan") {
            steps {
                echo "🕷️ Scan DAST avec OWASP ZAP..."
                sh '''
                    mkdir -p $(pwd)/zap-reports
                    docker run --rm \
                        -v $(pwd)/zap-reports:/zap/wrk/:rw \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t https://100.57.116.52 \
                        -r zap-report.html \
                        -I \
                        -z "-config network.connection.timeoutInSecs=60"
                '''
                echo "✅ Scan DAST terminé"
            }
        }
    }
    
    post {
        always {
            publishHTML target: [
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'dependency-check-report.html',
                reportName: 'OWASP Dependency-Check'
            ]
            
            publishHTML target: [
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'trivy-fs-report.html',
                reportName: 'Trivy Security Report'
            ]

            publishHTML target: [
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'zap-reports',
                reportFiles: 'zap-report.html',
                reportName: 'ZAP DAST Report'
            ]
            
            echo "🧹 Nettoyage..."
            cleanWs()
        }
        success {
            echo "🎉 SUCCÈS !"
            slackSend(
                webhookUrl: 'https://hooks.slack.com/services/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',
                channel: 'tous-jenkins-builds',
                color: 'good',
                message: """✅ *Build #${BUILD_NUMBER} — SUCCÈS* | *Job* : ${JOB_NAME} | *Durée* : ${currentBuild.durationString} | *App* : https://100.57.116.52 | *Logs* : ${BUILD_URL}console"""
            )
        }
        failure {
            echo "❌ ÉCHEC !"
            slackSend(
                webhookUrl: 'https://hooks.slack.com/services/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',
                channel: 'tous-jenkins-builds',
                color: 'danger',
                message: """❌ *Build #${BUILD_NUMBER} — ÉCHEC* | *Job* : ${JOB_NAME} | *Durée* : ${currentBuild.durationString} | Logs : ${BUILD_URL}console"""
            )
        }
    }
}
