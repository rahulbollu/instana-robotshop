pipeline {
  agent {
    docker {
      image 'rahulbollu/maven-rahul-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }

  parameters {
    string(name: 'DOCKER_IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag')
    string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git branch to build from')
  }

  environment {
    REPO_URL        = "https://github.com/rahulbollu/instana-robotshop.git"
    IMAGE_NAME      = "rahulbollu/cart"
    SONAR_HOST      = credentials('sonarqube-url')
    SONAR_TOKEN     = credentials('sonarqube-token')
    DOCKER_CREDS = credentials('docker-hub-creds')
    // DOCKER_USERNAME = credentials('docker-user')
    // DOCKER_PASSWORD = credentials('docker-pass')
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
          echo "sonar.projectKey=cart-service" > sonar-project.properties
          echo "sonar.sources=." >> sonar-project.properties
          echo "sonar.exclusions=**/*.java" >> sonar-project.properties
          
          echo "sonar.host.url=http://sonarqube:9000" >> sonar-project.properties
          echo "sonar.login=${SONAR_TOKEN}" >> sonar-project.properties

          docker run --rm \
            --network sonar-network \
            -v "$(pwd)":/usr/src -w /usr/src \
            sonarsource/sonar-scanner-cli sonar-scanner
        '''
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        echo "🐳 Building Docker image and pushing to Docker Hub"
        sh '''
          cd cart
          docker build -t ${IMAGE_NAME}:${DOCKER_IMAGE_TAG} .
          echo "${DOCKER_CREDS_PSW}" | docker login -u "${DOCKER_CREDS_USR}" --password-stdin
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

          # Check if the EKS cluster exists, if not, create it
          if ! aws eks describe-cluster --region us-west-2 --name robot-cluster > /dev/null 2>&1; then
               echo "🔧 EKS cluster 'robot-cluster' does not exist. Creating..."
          eksctl create cluster \
            --name robot-cluster \
            --region us-east-1 \
            --nodes 2 \
            --node-type t3.medium \
            --with-oidc \
            --managed
          else
            echo "✅ EKS cluster 'robot-cluster' already exists. Skipping creation."
          fi
          
          # Update kubeconfig for kubectl/helm access
          aws eks update-kubeconfig --region us-east-1 --name robot-cluster

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