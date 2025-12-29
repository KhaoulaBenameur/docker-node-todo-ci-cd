pipeline {
  agent any

  environment {
    DOCKERHUB_REPO = "khaoulabenameur/todo-app"
    K8S_NAMESPACE  = "mini-project"
    DEPLOYMENT     = "todo-app"
    CONTAINER_NAME = "todo-app"
    IMAGE_TAG      = "${BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        sh """
          docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
        """
      }
    }

    stage('Login & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh """
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh """
          kubectl get ns ${K8S_NAMESPACE} || kubectl create ns ${K8S_NAMESPACE}
          kubectl apply -n ${K8S_NAMESPACE} -f k8s/
          kubectl set image deployment/${DEPLOYMENT} ${CONTAINER_NAME}=${DOCKERHUB_REPO}:${IMAGE_TAG} -n ${K8S_NAMESPACE}
          kubectl rollout status deployment/${DEPLOYMENT} -n ${K8S_NAMESPACE}
          kubectl get pods -n ${K8S_NAMESPACE}
          kubectl get svc -n ${K8S_NAMESPACE}
        """
      }
    }
  }
}
