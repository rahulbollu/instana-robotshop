pipeline {
  agent any

  parameters {
    string(name: 'DOCKER_IMAGE_TAG', defaultValue: 'latest', description: 'Tag for Docker Image')
    string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git Branch to Build')
  }

  environment {
    SONAR_HOST      = credentials('sonarqube-url')      // Secret Text
    SONAR_TOKEN     = credentials('sonarqube-token')    // Secret Text
    DOCKER_USERNAME = credentials('docker-user')        // Username
    DOCKER_PASSWORD = credentials('docker-pass')        // Password
    GITHUB_TOKEN    = credentials('github-token')       // Secret Text
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key')   // Secret Text
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')   // Secret Text

    REPO_URL    = "https://github.com/rahulbollu/instana-robotshop.git"
    IMAGE_NAME  = "rahulbollu/cart-service"
    CLUSTER     = "three-tier-cluster"
    REGION      = "us-east-1"
    NAMESPACE   = "robot-shop"
  }

  stages {
    stage('Clone Repository') {
      steps {
        echo "✅ Cloning branch: ${params.GIT_BRANCH}"
        git branch: "${params.GIT_BRANCH}", url: "${env.REPO_URL}"
      }
    }

    stage('Build Cart Service') {
      steps {
        echo "🔧 Building cart service..."
        sh '''
          cd cart
          npm install
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        echo "🔍 Running static code analysis..."
        sh '''
          cd cart
          sonar-scanner \
            -Dsonar.projectKey=cart-service \
            -Dsonar.sources=. \
            -Dsonar.host.url=${SONAR_HOST} \
            -Dsonar.login=${SONAR_TOKEN}
        '''
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        echo "🐳 Building and pushing Docker image..."
        sh '''
          cd cart
          docker build -t ${IMAGE_NAME}:${DOCKER_IMAGE_TAG} .
          echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USERNAME}" --password-stdin
          docker push ${IMAGE_NAME}:${DOCKER_IMAGE_TAG}
        '''
      }
    }

    stage('Deploy to EKS via Helm') {
      steps {
        echo "🚀 Deploying to EKS..."
        sh '''
          export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
          export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}

          aws eks update-kubeconfig --name ${CLUSTER} --region ${REGION}

          helm upgrade --install cart ./EKS/helm \
            --namespace ${NAMESPACE} \
            --create-namespace \
            --set image.repository=${IMAGE_NAME} \
            --set image.tag=${DOCKER_IMAGE_TAG}
        '''
      }
    }
  }

  post {
    success {
      echo "✅ CI/CD Pipeline for Cart Service Completed Successfully!"
    }
    failure {
      echo "❌ CI/CD Failed. Please check the logs."
    }
  }
}
