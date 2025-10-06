pipeline {
  agent any

  environment {
    HELM_RELEASE = "myapp"
    HELM_CHART_PATH = "helm"
    KUBE_NAMESPACE = "default"
    KUBECONFIG_CREDENTIALS_ID = "kubeconfig-file" // Jenkins Secret file containing your Minikube kubeconfig
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set Image Tag') {
      steps {
        script {
          IMAGE_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.IMAGE_TAG = IMAGE_TAG
          echo "Using image tag: ${env.IMAGE_TAG}"
        }
      }
    }

    stage('Build Docker Image inside Minikube') {
      steps {
        sh '''
          # Point Docker CLI to Minikube’s Docker daemon
          eval $(minikube docker-env)
          docker build -t myapp:${IMAGE_TAG} .
        '''
      }
    }

    stage('Deploy to Minikube (Helm)') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
          sh '''
            export KUBECONFIG=${KUBECONFIG}
            helm repo update || true
            helm upgrade --install ${HELM_RELEASE} ${HELM_CHART_PATH} \
              --namespace ${KUBE_NAMESPACE} \
              --create-namespace \
              --set image.repository=myapp \
              --set image.tag=${IMAGE_TAG} \
              --set image.pullPolicy=IfNotPresent
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployment successful! Access with: minikube service ${HELM_RELEASE}-myapp"
    }
    failure {
      echo "❌ Build/Deploy failed. Check logs."
    }
  }
}
