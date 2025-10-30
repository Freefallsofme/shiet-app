pipeline {
    agent {
        kubernetes {
            label 'docker-agent'
            defaultContainer 'docker'
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: docker
                image: docker:24.0-dind
                command: ["sleep", "infinity"]
                securityContext:
                  privileged: true
                volumeMounts:
                - name: docker-sock
                  mountPath: /var/run/docker.sock
              - name: jnlp
                image: jenkins/inbound-agent:latest
                volumeMounts:
                - name: workspace-volume
                  mountPath: /home/jenkins/agent
              volumes:
              - name: docker-sock
                hostPath:
                  path: /var/run/docker.sock
              - name: workspace-volume
                emptyDir: {}
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
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def dockerImage = docker.build("${DOCKER_IMAGE}:${IMAGE_TAG}", ".")
                    dockerImage.tag("latest")
                    env.BUILT_IMAGE = dockerImage.id
                }
            }
        }

        stage('Test') {
            steps {
                container('docker') {
                    sh '''
                    apk add --no-cache python3 py3-pip
                    python3 -m pip install -r requirements.txt
                    python3 -m unittest discover tests
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        def img = docker.image("${DOCKER_IMAGE}:${IMAGE_TAG}")
                        img.push()
                        img.push('latest')
                    }
                }
            }
        }

        stage('Deploy to K8s') {
            steps {
                container('docker') {
                    sh '''
                    apk add --no-cache curl
                    curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
                    chmod +x kubectl
                    mv kubectl /usr/bin/

                    sed -i "s|image: .*|image: ${DOCKER_IMAGE}:latest|g" k8s/deployment.yaml
                    kubectl apply -f k8s/deployment.yaml -n ${K8S_NAMESPACE}
                    kubectl apply -f k8s/service.yaml -n ${K8S_NAMESPACE}
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker system prune -f || true'
        }
        success {
            echo 'CI/CD Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs.'
        }
    }
}
