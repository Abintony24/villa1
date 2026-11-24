pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'abintony/villa'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Semgrep SAST') {
            steps {
                sh 'semgrep scan --config p/security-audit .'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh 'trivy fs --scanners vuln,misconfig,secret .'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                    docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image ${DOCKER_IMAGE}:${BUILD_NUMBER}'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        docker.image("${DOCKER_IMAGE}:${BUILD_NUMBER}").push()
                        docker.image("${DOCKER_IMAGE}:latest").push()
                    }
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        sed -i "s|image: .*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|" k8s/deploy.yml

                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins"

                        git add k8s/deploy.yml
                        git commit -m "Update Villa image to ${BUILD_NUMBER}" || true

                        AUTH=$(printf "x-access-token:%s" "$GITHUB_TOKEN" | base64 -w0)
                        git -c http.extraheader="AUTHORIZATION: basic $AUTH" push origin HEAD:main
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
