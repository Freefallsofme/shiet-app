pipeline {
    agent {
        kubernetes {
            label 'docker-minikube'
            defaultContainer 'docker'
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: docker
                image: docker:24.0-cli
                command: ["sleep", "infinity"]
                volumeMounts:
                - name: docker-sock
                  mountPath: /var/run/docker.sock
              - name: jnlp
                image: jenkins/inbound-agent:latest
              volumes:
              - name: docker-sock
                hostPath:
                  path: /var/run/docker.sock
                  type: Socket
            '''
        }
    }

    environment {
        DOCKER_IMAGE = 'atatara/shiet-app'
        IMAGE_TAG    = "${BUILD_NUMBER}"
        K8S_NAMESPACE = 'default'
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build') {
            steps {
                sh '''
                docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} ${DOCKER_IMAGE}:latest
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                apk add --no-cache python3 py3-pip
                python3 -m pip install -r requirements.txt --break-system-packages
                python3 -m unittest discover tests
                '''
            }
        }

        stage('Push') {
            steps {
                withDockerRegistry([
                    credentialsId: 'docker-hub-credentials',
                    url: 'https://index.docker.io/v1/'
                ]) {
                    sh '''
                    docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                    docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                apk add --no-cache curl
                curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                chmod +x kubectl
                mv kubectl /usr/bin/

                sed -i "s|image: .*|image: ${DOCKER_IMAGE}:latest|g" k8s/deployment.yaml
                kubectl apply -f k8s/deployment.yaml -n ${K8S_NAMESPACE}
                kubectl apply -f k8s/service.yaml -n ${K8S_NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Flask deployed to K8s!'
        }
        failure {
            echo 'FAILURE!'
        }
        always {
            sh 'docker system prune -f || true'
        }
    }
}
