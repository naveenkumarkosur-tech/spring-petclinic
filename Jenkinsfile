pipeline {

    agent any

    tools {
        jdk 'JDK21'
        maven 'Maven3'
    }

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '139929688131'

        // IMPORTANT:
        // This is the actual ECR repository created by Terraform
        ECR_REPO = 'tfecr'

        IMAGE_NAME = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        IMAGE_TAG  = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Source') {
            steps {
                echo "Checking out source code..."
                checkout scm
            }
        }

        stage('Check Java Environment') {
            steps {
                sh '''
                    echo "=============================="
                    echo "JAVA_HOME=$JAVA_HOME"
                    echo "=============================="

                    java -version

                    echo "=============================="
                    javac -version

                    echo "=============================="
                    mvn -version

                    echo "=============================="
                    which java

                    echo "=============================="
                    which javac

                    echo "=============================="
                    readlink -f $(which java)

                    echo "=============================="
                    readlink -f $(which javac)

                    echo "=============================="
                    env | grep JAVA || true
                '''
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

        stage('Debug Trivy') {
            steps {
                sh '''
                    echo "=============================="
                    echo "User"
                    whoami

                    echo "=============================="
                    echo "Workspace"
                    pwd

                    echo "=============================="
                    echo "Files"
                    ls -la

                    echo "=============================="
                    echo "Trivy Location"
                    which trivy

                    echo "=============================="
                    echo "Trivy Version"
                    trivy --version
                '''
            }
        }

        stage('Trivy File Scan') {
            steps {
                sh '''
                    trivy fs .
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "Building Docker image:"
                    echo "${IMAGE_NAME}:${IMAGE_TAG}"

                    docker build \
                      -t "${IMAGE_NAME}:${IMAGE_TAG}" .
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image "${IMAGE_NAME}:${IMAGE_TAG}"
                '''
            }
        }

        stage('AWS Identity Check') {
            steps {
                sh '''
                    echo "AWS CLI:"
                    /usr/local/bin/aws --version

                    echo "AWS Identity:"
                    /usr/local/bin/aws sts get-caller-identity
                '''
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                sh '''
                    echo "Logging in to Amazon ECR..."

                    /usr/local/bin/aws ecr get-login-password \
                      --region "${AWS_REGION}" | \
                    docker login \
                      --username AWS \
                      --password-stdin \
                      "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                '''
            }
        }

        stage('Verify ECR Repository') {
            steps {
                sh '''
                    echo "Checking ECR repository..."

                    /usr/local/bin/aws ecr describe-repositories \
                      --repository-names "${ECR_REPO}" \
                      --region "${AWS_REGION}"
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    echo "Pushing image:"
                    echo "${IMAGE_NAME}:${IMAGE_TAG}"

                    docker push "${IMAGE_NAME}:${IMAGE_TAG}"
                '''
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar',
                                 fingerprint: true
            }
        }
    }

    post {

        success {
            echo 'CI Pipeline Completed Successfully'
            echo "Docker Image: ${IMAGE_NAME}:${IMAGE_TAG}"
        }

        failure {
            echo 'CI Pipeline Failed'
        }
    }
}
