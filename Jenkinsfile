pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'karthi051020/jenkinsdocker'
        SONAR_PROJECT  = 'SonarQube'
    }

    stages {

        stage('Build & Test') {
            steps {
                sh 'pip install -r requirements.txt --quiet'
                sh 'python -m pytest test_app.py -v'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT} \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=${SONAR_HOST_URL} \
                          -Dsonar.login=${SONAR_AUTH_TOKEN}
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${DOCKERHUB_REPO}:${BUILD_NUMBER} ."
                sh "docker tag ${DOCKERHUB_REPO}:${BUILD_NUMBER} ${DOCKERHUB_REPO}:latest"
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    sh "docker push ${DOCKERHUB_REPO}:${BUILD_NUMBER}"
                    sh "docker push ${DOCKERHUB_REPO}:latest"
                }
            }
        }

        stage('Deploy') {
            steps {
                // For local deploy:
                sh "docker stop app || true && docker rm app || true"
                sh "docker run -d --name app -p 5000:5000 ${DOCKERHUB_REPO}:latest"

                // For EC2 deploy — uncomment and set your EC2 IP:
                // sshagent(['ec2-ssh']) {
                //     sh """
                //         ssh -o StrictHostKeyChecking=no ubuntu@<EC2-IP> \
                //         'docker pull ${DOCKERHUB_REPO}:latest && \
                //          docker stop app || true && docker rm app || true && \
                //          docker run -d --name app -p 5000:5000 ${DOCKERHUB_REPO}:latest'
                //     """
                // }
            }
        }
    }

    post {
        failure { echo 'Pipeline failed — check logs above.' }
        success { echo 'Deployed successfully.' }
    }
}
