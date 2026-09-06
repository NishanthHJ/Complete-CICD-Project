
pipeline {
    agent any

    environment {
        IMAGE_NAME    = "nishanthhj02/fullstack:${BUILD_NUMBER}"
        AWS_REGION    = "us-east-1"
        CLUSTER_NAME  = "karunaadu"
        NAMESPACE     = "default"
        EKS_SERVER_URL = "https://BDEEE05DA910114810F68065B7708C9C.gr7.us-east-1.eks.amazonaws.com"
    }

    tools {
        jdk 'java-17'
        maven 'maven'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/NishanthHJ/Complete-CICD-Project.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                    mvn sonar:sonar \
                    -Dsonar.projectKey=devops \
                    -Dsonar.host.url=http://34.229.121.160:9000 \
                    -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html ${IMAGE_NAME}"
            }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                    echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh "docker push ${IMAGE_NAME}"
            }
        }

        stage('Update Kubeconfig') {
            steps {
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}"
            }
        }

        stage('Deploy To EKS') {
            steps {
                withKubeConfig(
                    credentialsId: 'kube',
                    namespace: "${NAMESPACE}",
                    serverUrl: "${EKS_SERVER_URL}",
                    clusterName: "${CLUSTER_NAME}"
                ) {
                    sh "sed -i 's|replace|${IMAGE_NAME}|g' deployment.yml"
                    sh "kubectl apply -f deployment.yml -n ${NAMESPACE}"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(
                    credentialsId: 'kube',
                    namespace: "${NAMESPACE}",
                    serverUrl: "${EKS_SERVER_URL}",
                    clusterName: "${CLUSTER_NAME}"
                ) {
                    sh "kubectl get pods -n ${NAMESPACE}"
                    sh "kubectl get svc -n ${NAMESPACE}"
                }
            }
        }
    }

    post {
        always {
            emailext(
                subject: "${JOB_NAME} - Build #${BUILD_NUMBER} - ${currentBuild.currentResult}",
                body: """
                Build Status: ${currentBuild.currentResult}

                Job: ${JOB_NAME}
                Build Number: ${BUILD_NUMBER}

                Console Output:
                ${BUILD_URL}
                """,
                to: 'hjnishanth6@gmail.com',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}