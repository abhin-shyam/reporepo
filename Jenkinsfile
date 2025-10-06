pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-cred')  // Jenkins stored creds (username+password/token)
        DOCKERHUB_REPO = "abhinshyam/myapp"                   // your DockerHub repo
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/abhin-shyam/reporepo.git'
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    IMAGE_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    echo "Using image tag: ${IMAGE_TAG}"
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    sh """
                        echo "${DOCKERHUB_CREDENTIALS_PSW}" | docker login -u "${DOCKERHUB_CREDENTIALS_USR}" --password-stdin
                        docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                        docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy to Minikube (Helm)') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=$HOME/.kube/config
                        helm upgrade --install myapp ./helm/myapp \
                            --set image.repository=${DOCKERHUB_REPO} \
                            --set image.tag=${IMAGE_TAG}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build and deploy successful"
        }
        failure {
            echo "❌ Build/Deploy failed. Check logs."
        }
    }
}
