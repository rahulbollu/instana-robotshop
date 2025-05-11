pipeline {
  agent any

  parameters {
    string(name: 'DOCKER_IMAGE_TAG', defaultValue: 'latest', description: 'Tag for Docker Image')
    string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git Branch to Build')
  }

  environment {
    SONAR_HOST      = credentials('sonarqube-url')      // Secret Text: http://<IP>:9000
    SONAR_TOKEN     = credentials('sonarqube-token')    // Secret Text
    DOCKER_USERNAME = credentials('docker-user')        // Username/Password
    DOCKER_PASSWORD = credentials('docker-pass')
    GITHUB_TOKEN    = credentials('github-token')       // Secret Text: GitHub PAT
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

    stage('Build Cart Service') {
      steps {
        echo "🔧 Building cart service..."
        sh '''
          cd cart
          mvn clean package
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        echo "🔍 Running static code analysis..."
        sh '''
          cd cart
          mvn sonar:sonar \
            -Dsonar.projectKey=cart-service \
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
          git push https://${GITHUB_TOKEN}@github.com/rahulbollu/Jenkins-Zero-To-Hero HEAD:${GIT_BRANCH}
        '''
      }
    }
  }

  post {
    success {
      echo "✅ CI/CD Pipeline for Cart Service Completed!"
    }
    failure {
      echo "❌ CI/CD Failed"
    }
  }
}
