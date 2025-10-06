pipeline {
  agent any

  environment {
    DOCKER_REPO = "yourdockerhubuser/myapp"   // update
    HELM_RELEASE = "myapp"
    HELM_CHART_PATH = "helm"
    KUBE_NAMESPACE = "default"
    // Important: create Jenkins credentials with these IDs and update below:
    DOCKERHUB_CREDENTIALS_ID = "dockerhub-creds"
    KUBECONFIG_CREDENTIALS_ID = "kubeconfig-file"  // Jenkins "Secret file" containing kubeconfig
    JIRA_CREDENTIALS_ID = "jira-creds" // username/password (username + APIToken)
    // optional: GITHUB_TOKEN to comment/mark PRs
    GITHUB_CREDENTIALS_ID = "github-token"
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

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${DOCKER_REPO}:${IMAGE_TAG} ."
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
            docker push ${DOCKER_REPO}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Deploy to Kubernetes (Helm)') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
          sh '''
            export KUBECONFIG=${KUBECONFIG}
            helm repo update || true
            helm upgrade --install ${HELM_RELEASE} ${HELM_CHART_PATH} \
              --namespace ${KUBE_NAMESPACE} \
              --set image.repository=${DOCKER_REPO} \
              --set image.tag=${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Notify JIRA') {
      steps {
        script {
          // Attempt to extract a Jira key from commit message (e.g. ABC-123)
          JIRA_KEY = sh(script: "git log -1 --pretty=%B | grep -o -E '[A-Z]+-[0-9]+' | head -1 || true", returnStdout: true).trim()
          if (JIRA_KEY) {
            withCredentials([usernamePassword(credentialsId: "${JIRA_CREDENTIALS_ID}", usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_TOKEN')]) {
              sh """
                curl -u $JIRA_USER:$JIRA_TOKEN -X POST -H 'Content-Type: application/json' \
                  --data '{ "body": "Build & deploy successful: ${DOCKER_REPO}:${IMAGE_TAG} deployed to ${KUBE_NAMESPACE} (Jenkins build ${env.BUILD_URL})" }' \
                  https://<YOUR_JIRA_DOMAIN>.atlassian.net/rest/api/2/issue/${JIRA_KEY}/comment
              """
            }
          } else {
            echo "No JIRA key found in commit message — skipping JIRA update."
          }
        }
      }
    }
  }

  post {
    failure {
      script {
        // Optionally update Jira with failure info
        JIRA_KEY = sh(script: "git log -1 --pretty=%B | grep -o -E '[A-Z]+-[0-9]+' | head -1 || true", returnStdout: true).trim()
        if (JIRA_KEY) {
          withCredentials([usernamePassword(credentialsId: "${JIRA_CREDENTIALS_ID}", usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_TOKEN')]) {
            sh """
              curl -u $JIRA_USER:$JIRA_TOKEN -X POST -H 'Content-Type: application/json' \
                --data '{ "body": "Build or deploy failed for commit ${env.GIT_COMMIT ?: 'unknown'}. See Jenkins: ${env.BUILD_URL}" }' \
                https://<YOUR_JIRA_DOMAIN>.atlassian.net/rest/api/2/issue/${JIRA_KEY}/comment
            """
          }
        } else {
          echo "No JIRA key found — cannot update Jira on failure."
        }
      }
    }
  }
}
