pipeline {
  agent { label "${env.BRANCH_NAME}-agent" }
  environment {
    ECR_URL = credentials("ecr-url-${env.BRANCH_NAME}")       // e.g. 'xxxx.dkr.ecr.region.amazonaws.com'
    AWS_CREDS = "aws-${env.BRANCH_NAME}"
    KUBECONFIG_ID = "kubeconfig-${env.BRANCH_NAME}"
    APP_NAME = "my-java-app"
    DOCKER_IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    NAMESPACE = "default"                                     // or 'dev', 'stage', etc.
    HELM_RELEASE = "${APP_NAME}-${env.BRANCH_NAME}"
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
          sh "docker build -t $ECR_URL/$APP_NAME:$DOCKER_IMAGE_TAG ."
        }
      }
    }
    stage('Login and Push to ECR') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDS}"]]) {
          sh """
            aws ecr get-login-password --region <aws-region> | docker login --username AWS --password-stdin $ECR_URL
            docker push $ECR_URL/$APP_NAME:$DOCKER_IMAGE_TAG
          """
        }
      }
    }
    stage('Deploy with Helm') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_ID}", variable: 'KUBECONFIG')]) {
          sh '''
            export KUBECONFIG=$KUBECONFIG
            # Upgrade Helm release; set the new image tag automatically
            helm upgrade --install $HELM_RELEASE ./helm-chart-dir \
              --namespace $NAMESPACE \
              --set image.repository=$ECR_URL/$APP_NAME \
              --set image.tag=$DOCKER_IMAGE_TAG
          '''
        }
      }
    }
  }
}