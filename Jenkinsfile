pipeline {
    agent any

    environment {

        SONAR_TOKEN = credentials('sonar-auth-token')
        IMAGE_NAME = 'lucifer1124/springboot-k8s-app'
        REGISTRY_CREDS = 'docker-cred' 
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                echo 'Compiling Spring Boot application....'

                sh 'mvn clean test package -DskipTests'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    echo 'Building image using scripted docker syntax...'
                    def customImage = docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")

                    echo 'Logging in and pushing build tags to Docker Hub...'
                    docker.withRegistry('https://index.docker.io/v1/', "${REGISTRY_CREDS}") {
                        customImage.push()
                        customImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Executing rolling update to Kubernetes Cluster...'

                withCredentials([file(credentialsId: 'k8s-config', variable: 'KUBECONFIG')]) {

                    sh "sed -i 's|LUCIFER_IMAGE_PLACEHOLDER|${IMAGE_NAME}:${BUILD_NUMBER}|g' k8s/deployment.yaml"                 

                    sh "kubectl apply -f k8s/deployment.yaml --kubeconfig=\$KUBECONFIG"
                    sh "kubectl apply -f k8s/service.yaml --kubeconfig=\$KUBECONFIG"
                    sh "kubectl rollout status deployment/springboot-deployment --kubeconfig=\$KUBECONFIG"
                }
            }
        }
    }

    post {
        always {
            echo 'Executing cleanup post-actions...'
            cleanWs()
            sh "docker rmi ${IMAGE_NAME}:${BUILD_NUMBER} || true"
            sh "docker rmi ${IMAGE_NAME}:latest || true"
        }
    }
}