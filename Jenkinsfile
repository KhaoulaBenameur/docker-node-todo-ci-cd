pipeline {
  agent any

  environment {
    // Docker
    DOCKERHUB_REPO = "khaoulabenameur/todo-app"
    IMAGE_TAG      = "${BUILD_NUMBER}"

    // Kubernetes / EKS
    AWS_REGION     = "us-east-1"
    EKS_CLUSTER    = "mini-project-eks"
    K8S_NAMESPACE  = "mini-project"
    DEPLOYMENT     = "todo-app"
    CONTAINER_NAME = "todo-app"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          set -e
          docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
        '''
      }
    }

    stage('Login & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            set -e
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin

            # retry push 3 times
            for i in 1 2 3; do
              echo "Attempt $i: pushing image ${DOCKERHUB_REPO}:${IMAGE_TAG} ..."
              docker push ${DOCKERHUB_REPO}:${IMAGE_TAG} && break
              echo "Push failed, retrying in 10s..."
              sleep 10
            done
          '''
        }
      }
    }

    stage('Deploy to EKS') {
      agent {
        docker {
          image 'amazon/aws-cli:2.15.0'
          args '-u root:root --entrypoint=""'
        }
      }
      steps {
        withCredentials([
          string(credentialsId: 'aws_access_key_id', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws_secret_access_key', variable: 'AWS_SECRET_ACCESS_KEY'),
          string(credentialsId: 'aws_session_token', variable: 'AWS_SESSION_TOKEN')
        ]) {
          sh '''
            set -e

            echo "== Installing tools (kubectl) =="
            yum -y install curl tar gzip >/dev/null

            ARCH="$(uname -m)"
            if [ "$ARCH" = "x86_64" ]; then ARCH="amd64"; fi
            if [ "$ARCH" = "aarch64" ]; then ARCH="arm64"; fi

            KVER="$(curl -sL https://dl.k8s.io/release/stable.txt)"
            curl -sLO "https://dl.k8s.io/release/${KVER}/bin/linux/${ARCH}/kubectl"
            install -m 0755 kubectl /usr/local/bin/kubectl

            echo "== Who am I? (AWS) =="
            aws sts get-caller-identity

            echo "== Update kubeconfig for EKS =="
            aws eks update-kubeconfig --region "${AWS_REGION}" --name "${EKS_CLUSTER}"

            echo "== Namespace =="
            kubectl get ns "${K8S_NAMESPACE}" >/dev/null 2>&1 || kubectl create ns "${K8S_NAMESPACE}"

            echo "== Apply manifests =="
            kubectl apply -n "${K8S_NAMESPACE}" -f k8s/

            echo "== Update image =="
            kubectl set image deployment/${DEPLOYMENT} ${CONTAINER_NAME}=${DOCKERHUB_REPO}:${IMAGE_TAG} -n "${K8S_NAMESPACE}"

            echo "== Rollout status =="
            kubectl rollout status deployment/${DEPLOYMENT} -n "${K8S_NAMESPACE}"

            echo "== Result =="
            kubectl get pods -n "${K8S_NAMESPACE}"
            kubectl get svc  -n "${K8S_NAMESPACE}"
          '''
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline finished."
    }
  }
}



// pipeline {
//   agent any

//   environment {
//     DOCKERHUB_REPO = "khaoulabenameur/todo-app"
//     K8S_NAMESPACE  = "mini-project"
//     DEPLOYMENT     = "todo-app"
//     CONTAINER_NAME = "todo-app"
//     IMAGE_TAG      = "${BUILD_NUMBER}"
//   }

//   stages {
//     stage('Checkout') {
//       steps {
//         checkout scm
//       }
//     }

//     stage('Build Docker Image') {
//       steps {
//         sh """
//           docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
//         """
//       }
//     }

//     stage('Login & Push') {
//       steps {
//         withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
//           sh """
//             echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
//             docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
//           """
//         }
//       }
//     }

//     stage('Deploy to EKS') {
//       steps {
//         sh """
//           kubectl get ns ${K8S_NAMESPACE} || kubectl create ns ${K8S_NAMESPACE}
//           kubectl apply -n ${K8S_NAMESPACE} -f k8s/
//           kubectl set image deployment/${DEPLOYMENT} ${CONTAINER_NAME}=${DOCKERHUB_REPO}:${IMAGE_TAG} -n ${K8S_NAMESPACE}
//           kubectl rollout status deployment/${DEPLOYMENT} -n ${K8S_NAMESPACE}
//           kubectl get pods -n ${K8S_NAMESPACE}
//           kubectl get svc -n ${K8S_NAMESPACE}
//         """
//       }
//     }
//   }
// }
