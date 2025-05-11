pipeline {
  agent any

  parameters {
    string(name: 'DOCKER_IMAGE_TAG', defaultValue: 'latest', description: 'Tag for Docker Image')
    string(name: 'GIT_BRANCH', defaultValue: 'develop', description: 'Git Branch to Build')
  }

  environment {
    SONAR_HOST      = credentials('sonarqube-url')
    SONAR_TOKEN     = credentials('sonarqube-token')
    DOCKER_USERNAME = credentials('docker-user')
    DOCKER_PASSWORD = credentials('docker-pass')
    GITHUB_TOKEN    = credentials('github-token')
    REPO_URL        = "https://github.com/rahulbollu/instana-robotshop.git"
    IMAGE_NAME      = "rahulbollu/cart-service"
  }

  stages {

    stage('Clone Repository') {
      steps {
        echo "✅ Cloning branch: ${params.GIT_BRANCH}"
        git branch: "${params.GIT_BRANCH}", url: "${env.REPO_URL}"
      }
    }

    stage('Install Dependencies') {
      steps {
        echo "🔧 Installing Node dependencies..."
        sh '''
          cd cart
          npm install
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        echo "🔍 Running SonarQube Analysis..."
        sh '''
          cd cart
          sonar-scanner \
            -Dsonar.projectKey=cart-service \
            -Dsonar.sources=. \
            -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
            -Dsonar.host.url=${SONAR_HOST} \
            -Dsonar.login=${SONAR_TOKEN} || echo "SonarQube scan skipped (optional)"
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

    stage('Update Kubernetes Deployment') {
      steps {
        echo "✏️ Updating deployment file with image tag..."
        sh '''
          git config --global --add safe.directory `pwd`
          git config --global user.email "bvrahul3141@gmail.com"
          git config --global user.name "RahulBollu"
          sed -i "s/replaceImageTag/${DOCKER_IMAGE_TAG}/g" cart/k8s/deployment.yml
          git add cart/k8s/deployment.yml
          git commit -m "Update cart image to tag ${DOCKER_IMAGE_TAG}" || echo "No changes to commit"
          git push https://${GITHUB_TOKEN}@github.com/rahulbollu/instana-robotshop HEAD:${GIT_BRANCH}
        '''
      }
    }
  }

  post {
    success {
      echo "✅ CI/CD Pipeline for Cart Service Completed!"
    }
    failure {
      echo "❌ CI/CD Pipeline Failed"
    }
  }
}
