pipeline {
    agent { label "master" }
    environment {
        ECR_REGISTRY = "370639238640.dkr.ecr.us-east-1.amazonaws.com"
        APP_REPO_NAME= "bestcloud/spring-boot-app"
    }
    stages {
        stage('Build Docker Compile image') {
            steps {
                sh 'docker run --rm -v $HOME/.m2:/root/.m2 -v $WORKSPACE:/app -w /app maven:3.6.3-openjdk-8 mvn clean package'
                sh 'docker image ls'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build --force-rm -t "$ECR_REGISTRY/$APP_REPO_NAME:latest" .'
                sh 'docker image ls'
            }
        }
        stage('Push Image to ECR Repo') {
            steps {
                sh 'PATH="$PATH:/usr/local/bin"'
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }
        stage('Deploy') {
            steps {
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker pull "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
                sh 'docker run --name todo -dp 80:8080 "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }

    }
    post {
        always {
            echo 'Deleting all local images'
            sh 'docker image prune -af'
        }
    }
}