
pipeline {
    agent any
 
    environment {
        IMAGE_NAME   = "nishanth/fullstack:${GIT_COMMIT}"
        AWS_REGION   = "us-east-1"
        CLUSTER_NAME = "karunaadu"
        NAMESPACE    = "karunaadu"
        EKS_SERVER_URL = "https://BDEEE05DA910114810F68065B7708C9C.gr7.us-east-1.eks.amazonaws.com"
    }
 
    tools {
        jdk 'java-17'
        maven 'maven'
    }
 
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/NishanthHJ/Complete-CICD-Project.git'
            }
        }
 
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
 
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
 
        stage('sonarqube-stage') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                    mvn sonar:sonar \
                    -Dsonar.projectKey=devops \
                    -Dsonar.host.url=http://34.229.121.160:9000/ \
                    -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }
 
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    sh 'printenv'
                    sh "docker build -t ${IMAGE_NAME} ."
                }
            }
        }
 
        stage('Docker Image Scan') {
            steps {
                script {
                    sh "trivy image --format table -o trivy-image-report.html ${IMAGE_NAME}"
                }
            }
        }
 
        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin"
                    }
                }
            }
        }
 
        stage('Push Docker Image') {
            steps {
                script {
                    sh "docker push ${IMAGE_NAME}"
                }
            }
        }
 
        stage('Updating the Cluster') {
            steps {
                script {
                    sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}"
                }
            }
        }
 
        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'karunaadu', contextName: '', credentialsId: 'kube', namespace: "${NAMESPACE}", restrictKubeConfigAccess: false, serverUrl: "${EKS_SERVER_URL}") {
                    sh "sed -i 's|replace|${IMAGE_NAME}|g' deployment.yml"
                    sh "kubectl apply -f deployment.yml -n ${NAMESPACE}"
                }
            }
        }
 
        stage('Verify the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'karunaadu', contextName: '', credentialsId: 'kube', namespace: "${NAMESPACE}", restrictKubeConfigAccess: false, serverUrl: "${EKS_SERVER_URL}") {
                    sh "kubectl get pods -n ${NAMESPACE}"
                    sh "kubectl get svc -n ${NAMESPACE}"
                }
            }
        }
    }
 
    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'
 
                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """
 
                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'hjnishanth6@gmail.com',
                    from: 'hjnishanth6@gmail.com',
                    replyTo: 'nishanthhj7@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
 