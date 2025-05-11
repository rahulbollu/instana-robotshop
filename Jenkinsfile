pipeline {
  agent {
    docker {
      image 'rahulbollu/maven-rahul-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }

  parameters {
    string(name: 'DOCKER_IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag')
    string(name: 'GIT_BRANCH', defaultValue: 'develop', description: 'Git branch to build from')
  }

  environment {
    REPO_URL        = "https://github.com/rahulbollu/instana-robotshop.git"
    IMAGE_NAME      = "rahulbollu/cart"
    SONAR_HOST      = credentials('sonarqube-url')
    SONAR_TOKEN     = credentials('sonarqube-token')
    DOCKER_USERNAME = credentials('docker-user')
    DOCKER_PASSWORD = credentials('docker-pass')
    AWS_ACCESS_KEY  = credentials('aws-access-key')
    AWS_SECRET_KEY  = credentials('aws-secret-key')
    GITHUB_TOKEN    = credentials('github-token')
  }

  stages {

    stage('Clone Repository') {
      steps {
        echo "📥 Cloning branch: ${params.GIT_BRANCH}"
        git branch: "${params.GIT_BRANCH}", url: "${env.REPO_URL}"
      }
    }

    stage('Install Dependencies') {
      steps {
        echo "📦 Installing cart dependencies"
        sh '''
          cd cart
          npm install
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        echo "🔍 Running SonarQube Analysis"
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
        echo "🐳 Building Docker image and pushing to Docker Hub"
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
        echo "🚀 Deploying to EKS using Helm"
        sh '''
          export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY}
          export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_KEY}
          aws eks update-kubeconfig --region us-west-2 --name robot-cluster

          helm upgrade --install cart ./EKS/helm \
            --namespace robot-shop \
            --create-namespace \
            --set image.repo=${IMAGE_NAME} \
            --set image.version=${DOCKER_IMAGE_TAG}
        '''
      }
    }
  }

  post {
    success {
      echo "✅ CI/CD Pipeline completed successfully!"
    }
    failure {
      echo "❌ CI/CD Pipeline failed."
    }
  }
}