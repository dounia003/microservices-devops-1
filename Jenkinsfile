pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'dounia03'
    }

    stages {
        stage('SCA with OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '''--format HTML''', odcInstallation: 'DP-Check'
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
                        -Dsonar.projectKey=newsread-microservice-application \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=${env.SONAR_HOST_URL} \
                        -Dsonar.login=${env.SONAR_AUTH_TOKEN}
                    """
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh 'docker build -t ${DOCKERHUB_USER}/newsread-customize customize-service/'
                    sh 'docker build -t ${DOCKERHUB_USER}/newsread-news news-service/'
                }
            }
        }

        stage('Containerize And Test') {
            steps {
                script {
                    sh 'docker run -d --name customize-service -e FLASK_APP=run.py ${DOCKERHUB_USER}/newsread-customize && sleep 10 && docker logs customize-service && docker stop customize-service'
                    sh 'docker run -d --name news-service -e FLASK_APP=run.py ${DOCKERHUB_USER}/newsread-news && sleep 10 && docker logs news-service && docker stop news-service'
                }
            }
        }

        stage('Push Images To DockerHub') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'DockerHubPass', variable: 'DOCKERHUB_PASS')]) {
                        sh 'docker login -u ${DOCKERHUB_USER} --password ${DOCKERHUB_PASS}'
                    }
                    sh 'docker push ${DOCKERHUB_USER}/newsread-news'
                    sh 'docker push ${DOCKERHUB_USER}/newsread-customize'
                }
            }
        }

        // Optionnel : analyse vulnérabilités avec Trivy
        // stage('Trivy scan on Docker images') {
        //     steps {
        //         sh 'TMPDIR=/home/jenkins trivy image ${DOCKERHUB_USER}/newsread-news:latest'
        //         sh 'TMPDIR=/home/jenkins trivy image ${DOCKERHUB_USER}/newsread-customize:latest'
        //     }
        // }
    }

    post {
        always {
            sh 'docker rm -f news-service || true'
            sh 'docker rm -f customize-service || true'
        }
        success {
            sh 'docker logout'
        }
    }
}

