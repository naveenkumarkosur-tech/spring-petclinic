pipeline {

    agent any

    tools {
        jdk 'JDK21'
        maven 'Maven3'
    }

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '139929688131'

        // ECR repository name
        ECR_REPO       = 'tfecr'

        IMAGE_NAME     = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        
        // Creates a unique tag every time (e.g., build-62-202609011610)
        IMAGE_TAG      = "build-${BUILD_NUMBER}-${BUILD_TIMESTAMP}"
    }

    stages {

        stage('Checkout Source') {
            steps {
                echo "Checking out source code..."
                checkout scm
            }
        }

        stage('Compile') {
            steps {
                sh '''
                    mkdir -p target
                    mvn clean compile
                '''
            }
        }

        stage('Unit Test') {
            steps {
                sh '''
                    mkdir -p target
                    mvn test
                '''
            }
        }

        stage('Package') {
            steps {
                sh '''
                    mvn package -DskipTests
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.7.0.6970:sonar \
                          -Dsonar.projectKey=spring-petclinic
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trivy File Scan') {
            steps {
                sh 'trivy fs .'
            }
        }

        // NOTE: Use double quotes """ so Jenkins properly fills in ${IMAGE_NAME} and ${IMAGE_TAG}
        stage('Docker Build') {
            steps {
                sh """
                    echo "Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                    docker build -t "${IMAGE_NAME}:${IMAGE_TAG}" -t "${IMAGE_NAME}:latest" .
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                    trivy image "${IMAGE_NAME}:${IMAGE_TAG}"
                """
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                sh """
                    echo "Logging in to Amazon ECR..."
                    /usr/local/bin/aws ecr get-login-password --region "${AWS_REGION}" | \
                    docker login --username AWS --password-stdin "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                    echo "Pushing image: ${IMAGE_NAME}:${IMAGE_TAG}"
                    docker push "${IMAGE_NAME}:${IMAGE_TAG}"
                    docker push "${IMAGE_NAME}:latest"
                """
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }

    post {
        always {
            // Clean up local images after push to prevent disk space issues
            sh """
                docker rmi "${IMAGE_NAME}:${IMAGE_TAG}" "${IMAGE_NAME}:latest" || true
            """
        }
        success {
            echo "CI Pipeline Completed Successfully"
            echo "Docker Image Pushed: ${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo "CI Pipeline Failed"
        }
    }
}
