Jenkins for Docker Swarm:-


    pipeline {
      agent any
        environment {
            // Replace with your Docker Hub username and repository name
            SONAR_TOKEN = credentials('sonar-auth-token')
            IMAGE_NAME = 'lucifer1124/hello-swarm'
        }
    
        stages {
            stage('Checkout') {
                steps {
                    checkout scm
                }
            }
    
            stage('Build & Test') {
                steps {
                    // Compile and verify code locally before packaging into Docker
                    sh 'mvn clean test package -DskipTests'
                }
            }
    
            stage('Docker Build & Push') {
                steps {
                    script {
                        // Build image using Jenkins' unique build number as tag
                        def customImage = docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
    
                        // Log into Docker Hub and push the image
                        docker.withRegistry('https://index.docker.io/v1/', "${REGISTRY_CREDS}") {
                            customImage.push()
                            customImage.push('latest')
                        }
                    }
                }
            }
    
            stage('Deploy to Docker Swarm') {
                steps {
                    // If Jenkins is running on the Swarm manager, it deploys directly.
                    // If it is remote, wrap this step using an SSH agent or context.
                    sh '''
                        docker stack deploy -c docker-stack.yml hello_stack
                    '''
                }
            }
        }
    
        post {
            always {
                // Clean up workspace and local image cache to prevent disk bloating
                cleanWs()
                sh "docker rmi ${IMAGE_NAME}:${BUILD_NUMBER} || true"
            }
        }
  }
