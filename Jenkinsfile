pipeline {
    agent {
        dockerfile {
            filename 'Dockerfile.build'
            args '-v /root/.m2:/root/.m2 -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    //agent any

    environment {
        DOCKERHUB_AUTH = credentials('DockerHubCredentials')
        MYSQL_AUTH= credentials('MYSQL_AUTH')
        ID_DOCKER = "${DOCKERHUB_AUTH_USR}" // ou votre username directement
        IMAGE_NAME = "paymybuddy"
        IMAGE_TAG = "latest"
        SPRING_DATASOURCE_URL="jdbc:mysql://172.17.0.1:3306/db_paymybuddy"
        EXPOSE_PORT="8090"
        CONTAINER_SQL="container_sql"
        CONTAINER_APP="paymybuddy"
    }

    stages {
        /*stage('Test') {
            steps {
                sh 'mvn clean test'
            }

            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }*/

        stage('Package') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('SonarqubeServer') {
                    sh 'mvn sonar:sonar -s .m2/settings.xml'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 60, unit: 'SECONDS') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build and push IMAGE to docker registry') {
            steps {
                sh """
                    docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .
                    echo ${DOCKERHUB_AUTH_PSW} | docker login -u ${DOCKERHUB_AUTH_USR} --password-stdin
                    docker push ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

         stage ('Deploy in staging') {
            agent any
            environment {
                HOSTNAME_DEPLOY_STAGING = "184.73.142.22"
            }
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) { 
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -t rsa,dsa ${HOSTNAME_DEPLOY_STAGING} >> ~/.ssh/known_hosts
                        scp src/main/resources/database/create.sql centos@${HOSTNAME_DEPLOY_STAGING}:/home/centos/
                        command1="echo ${DOCKERHUB_AUTH_PSW} | docker login -u ${DOCKERHUB_AUTH_USR} --password-stdin"
                        command2="docker pull ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}"
                        command3="docker rm -f ${CONTAINER_APP} || echo 'app does not exist'"
                        command4="docker run -d --name ${CONTAINER_SQL} -p 3306:3306 -e MYSQL_ROOT_PASSWORD=${MYSQL_AUTH_PSW} -d mysql || echo 'database already up'"
                        command5="docker exec -i ${CONTAINER_SQL} sh -c 'exec mysql -u${MYSQL_AUTH_USR} -p${MYSQL_AUTH_PSW} ' < create.sql || echo 'database already exist'"
                        command6="docker run -d --name ${CONTAINER_APP} -e SPRING_DATASOURCE_USERNAME=${MYSQL_AUTH_USR} -e SPRING_DATASOURCE_PASSWORD=${MYSQL_AUTH_PSW} -e SPRING_DATASOURCE_URL=${SPRING_DATASOURCE_URL} -p ${EXPOSE_PORT}:8080 ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}"
                        ssh -t centos@${HOSTNAME_DEPLOY_STAGING} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=IMAGE_TAG \
                            -o SendEnv=DOCKERHUB_AUTH_USR \
                            -o SendEnv=DOCKERHUB_AUTH_PSW \
                            -o SendEnv=MYSQL_AUTH_USR \
                            -o SendEnv=MYSQL_AUTH_PSW \
                            -o SendEnv=SPRING_DATASOURCE_URL \
                            -o SendEnv=EXPOSE_PORT \
                            -o SendEnv=CONTAINER_APP \
                            -o SendEnv=CONTAINER_SQL \
                            -C "$command1 && $command2 && $command3 && $command4 && $command5 && $command6"
                    '''
                }
            }
        }

         stage ('Deploy in prod') {
            agent any
            environment {
                HOSTNAME_DEPLOY_PROD = "3.85.31.2"
            }
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) { 
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -t rsa,dsa ${HOSTNAME_DEPLOY_PROD} >> ~/.ssh/known_hosts
                        scp src/main/resources/database/create.sql centos@${HOSTNAME_DEPLOY_PROD}:/home/centos/
                        command1="echo ${DOCKERHUB_AUTH_PSW} | docker login -u ${DOCKERHUB_AUTH_USR} --password-stdin"
                        command2="docker pull ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}"
                        command3="docker rm -f ${CONTAINER_APP} || echo 'app does not exist'"
                        command4="docker run -d --name ${CONTAINER_SQL} -p 3306:3306 -e MYSQL_ROOT_PASSWORD=${MYSQL_AUTH_PSW} -d mysql || echo 'database already up'"
                        command5="docker exec -i ${CONTAINER_SQL} sh -c 'exec mysql -u${MYSQL_AUTH_USR} -p${MYSQL_AUTH_PSW} ' < create.sql || echo 'database already exist'"
                        command6="docker run -d --name ${CONTAINER_APP} -e SPRING_DATASOURCE_USERNAME=${MYSQL_AUTH_USR} -e SPRING_DATASOURCE_PASSWORD=${MYSQL_AUTH_PSW} -e SPRING_DATASOURCE_URL=${SPRING_DATASOURCE_URL} -p ${EXPOSE_PORT}:8080 ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}"
                        ssh -t centos@${HOSTNAME_DEPLOY_PROD} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=IMAGE_TAG \
                            -o SendEnv=DOCKERHUB_AUTH_USR \
                            -o SendEnv=DOCKERHUB_AUTH_PSW \
                            -o SendEnv=MYSQL_AUTH_USR \
                            -o SendEnv=MYSQL_AUTH_PSW \
                            -o SendEnv=SPRING_DATASOURCE_URL \
                            -o SendEnv=EXPOSE_PORT \
                            -o SendEnv=CONTAINER_APP \
                            -o SendEnv=CONTAINER_SQL \
                            -C "$command1 && $command2 && $command3 && $command4 && $command5 && $command6"
                    '''
                }
            }
        }

    }
}