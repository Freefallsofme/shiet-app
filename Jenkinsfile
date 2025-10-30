pipeline {
    agent {
    kubernetes {
	label 'docker-agent'
	defaultContainer 'docker'
	}
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                   sh 'docker build -t ${DOCKER-IMAGE}:$BUILD_NUMBER} .'
		}
                }
            }
        }
        stage('Test') {
            steps {
                sh 'pip install -r requirements.txt'
                sh 'python -m unittest discover tests'
            }
        }
        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        docker_image.push()
                        docker_image.push('latest')
                    }
                }
            }
        }
        stage('Deploy to K8s') {
            steps {
                script {
                    sh "sed -i 's/your-registry/flask-app:latest/g' k8s/deployment.yaml"  // Update image tag
                    sh "kubectl apply -f k8s/ -n ${K8S_NAMESPACE}"
                }
            }
        }
    }
}
