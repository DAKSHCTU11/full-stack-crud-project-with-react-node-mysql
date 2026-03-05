pipeline {
  agent any

  environment {
    AWS_REGION      = 'us-east-1'
    ECR_REGISTRY    = '360025768948.dkr.ecr.us-east-1.amazonaws.com'
    BACKEND_REPO    = 'crud-app-backend'
    FRONTEND_REPO   = 'crud-app-frontend'
    IMAGE_TAG       = "${env.BUILD_NUMBER}"
    GITOPS_DIR      = 'crud-app-gitops'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        echo "Building image tag: ${IMAGE_TAG}"
      }
    }

    stage('ECR Login') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-creds',
          accessKeyVariable: 'AWS_ACCESS_KEY_ID',
          secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
        ]]) {
          sh '''
            aws ecr get-login-password --region $AWS_REGION | \
              docker login --username AWS --password-stdin $ECR_REGISTRY
          '''
        }
      }
    }

    stage('Build Backend') {
      steps {
        sh '''
          docker build -t $ECR_REGISTRY/$BACKEND_REPO:$IMAGE_TAG ./backend
          docker tag $ECR_REGISTRY/$BACKEND_REPO:$IMAGE_TAG $ECR_REGISTRY/$BACKEND_REPO:latest
        '''
      }
    }

    stage('Build Frontend') {
      steps {
        sh '''
          docker build \
            --build-arg VITE_API_URL=http://backend-svc:3000 \
            -t $ECR_REGISTRY/$FRONTEND_REPO:$IMAGE_TAG ./frontend
          docker tag $ECR_REGISTRY/$FRONTEND_REPO:$IMAGE_TAG $ECR_REGISTRY/$FRONTEND_REPO:latest
        '''
      }
    }

    stage('Push Images to ECR') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-creds',
          accessKeyVariable: 'AWS_ACCESS_KEY_ID',
          secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
        ]]) {
          sh '''
            docker push $ECR_REGISTRY/$BACKEND_REPO:$IMAGE_TAG
            docker push $ECR_REGISTRY/$BACKEND_REPO:latest
            docker push $ECR_REGISTRY/$FRONTEND_REPO:$IMAGE_TAG
            docker push $ECR_REGISTRY/$FRONTEND_REPO:latest
          '''
        }
      }
    }

    stage('Update GitOps Repo') {
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
          sh '''
            git config --global user.email "jenkins@ci.com"
            git config --global user.name "Jenkins"

            rm -rf $GITOPS_DIR
            git clone https://DAKSHCTU11:$GITHUB_TOKEN@github.com/DAKSHCTU11/crud-app-gitops $GITOPS_DIR

            sed -i "s|image: $ECR_REGISTRY/$BACKEND_REPO:.*|image: $ECR_REGISTRY/$BACKEND_REPO:$IMAGE_TAG|g" $GITOPS_DIR/k8s/backend-deployment.yaml
            sed -i "s|image: $ECR_REGISTRY/$FRONTEND_REPO:.*|image: $ECR_REGISTRY/$FRONTEND_REPO:$IMAGE_TAG|g" $GITOPS_DIR/k8s/frontend-deployment.yaml

            cd $GITOPS_DIR
            git add .
            git diff --cached --quiet || git commit -m "Update image tags to $IMAGE_TAG"
            git push origin main
          '''
        }
      }
    }

    stage('Cleanup') {
      steps {
        sh '''
          docker rmi $ECR_REGISTRY/$BACKEND_REPO:$IMAGE_TAG || true
          docker rmi $ECR_REGISTRY/$FRONTEND_REPO:$IMAGE_TAG || true
        '''
      }
    }

  }

  post {
    success {
      echo '✅ Build done! ArgoCD will auto-deploy to EKS.'
    }
    failure {
      echo '❌ Build failed!'
    }
  }
}
