pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = '900110461895.dkr.ecr.us-east-1.amazonaws.com'
        CUSTOMIZE_IMAGE = "${ECR_REGISTRY}/devops-pfs/customize-service:latest"
        NEWS_IMAGE = "${ECR_REGISTRY}/devops-pfs/news-service:latest"
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('SCA - OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--format HTML',
                odcInstallation: 'DP-Check'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    scannerHome = tool 'SonarScanner'
                }
                withSonarQubeEnv('Sonarqube Server') {
                    sh """
                    ${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=newsread-microservices \
                    -Dsonar.sources=.
                    """
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                docker build -t customize-service:latest customize-service/
                docker build -t news-service:latest news-service/
                '''
            }
        }

        stage('Container Test') {
            steps {
                sh '''
                docker run -d --name customize-test \
                  -e FLASK_APP=run.py customize-service:latest
                sleep 10
                docker logs customize-test
                docker rm -f customize-test

                docker run -d --name news-test \
                  -e FLASK_APP=run.py news-service:latest
                sleep 10
                docker logs news-test
                docker rm -f news-test
                '''
            }
        }

        stage('Tag Images for ECR') {
            steps {
                sh '''
                docker tag customize-service:latest ${CUSTOMIZE_IMAGE}
                docker tag news-service:latest ${NEWS_IMAGE}
                '''
            }
        }

        stage('Login to AWS ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials'
                ]]) {
                    sh '''
                    aws ecr get-login-password --region ${AWS_REGION} |
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    '''
                }
            }
        }

        stage('Push Images to AWS ECR') {
            steps {
                sh '''
                docker push ${CUSTOMIZE_IMAGE}
                docker push ${NEWS_IMAGE}
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                echo "🔍 Trivy scan - customize-service"
                trivy image --severity HIGH,CRITICAL --exit-code 1 ${CUSTOMIZE_IMAGE}

                echo "🔍 Trivy scan - news-service"
                trivy image --severity HIGH,CRITICAL --exit-code 1 ${NEWS_IMAGE}
                '''
            }
        }
    }

    post {
        always {
            sh '''
            docker system prune -af || true
            '''
        }

        success {
            echo '✅ Pipeline executed successfully!'
        }

        failure {
            echo '❌ Pipeline failed – Check logs'
        }
    }
}
