pipeline {
    agent {
        dockerfile {
            filename 'Dockerfile.build'
            args 'FILLME'
        }
    }

    environment {
        DOCKERHUB_AUTH = credentials('DockerHubCredentials')
        ID_DOCKER = "${DOCKERHUB_AUTH_USR}" // ou votre username directement
    }

    stages {
        stage('Test') {
            steps {
                sh 'mvn clean test'
            }

            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 60, unit: 'SECONDS') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build and push IMAGE to Docker registry') {
            sh """
                docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .
                echo ${DOCKERHUB_AUTH_PSW} | docker login -u ${DOCKERHUB_AUTH_USR} --password-stdin
                docker push ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
            """
        }
    }
}