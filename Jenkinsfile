pipeline {

    agent any

    tools {
        jdk 'JDK21'
        maven 'Maven3'
    }

    environment {
        AWS_REGION = "ap-south-1"
        AWS_ACCOUNT_ID = "139929688131"   // Replace with your AWS Account ID
        ECR_REPO = "spring-petclinic"

        IMAGE_NAME = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Source') {
            steps {
                echo "Source code already checked out by Jenkins"
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
                mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
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
        whoami
        pwd
        ls -la
        which trivy
        trivy --version
        '''
    }
}
        stage('Trivy File Scan') {
            steps {
                sh 'trivy fs .'
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} .
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                    trivy image ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login \
                    --username AWS \
                    --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar'
            }
        }

    }

    post {

        success {
            echo 'CI Pipeline Completed Successfully'
        }

        failure {
            echo 'CI Pipeline Failed'
        }


    }

}
