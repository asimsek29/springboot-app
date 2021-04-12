### Project: dummy-spring-boot

This project aims to create full CI/CD Pipeline for microservice based applications using [dummy-spring-boot](https://gitlab.com/bcfmkubilay/dummy-spring-boot.git). Jenkins Server deployed on Elastic Compute Cloud (EC2) Instance is used as CI/CD Server to build pipelines.

## part1: Prepare Development Server Manually on EC2 Instance
Prepare development server manually on Amazon Linux 2 for developers, enabled with `Docker`,  `Docker-Compose`,  `Java 11`,  `Git`.
``` bash
#! /bin/bash
yum update -y
hostnamectl set-hostname springboot-app-server
amazon-linux-extras install docker -y
systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user
curl -L "https://github.com/docker/compose/releases/download/1.26.2/docker-compose-$(uname -s)-$(uname -m)" \
-o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
yum install git -y
yum install java-11-amazon-corretto -y
```

## Prepare GitHub Repository for the Project
* Connect to your Development Server via `ssh` and clone the petclinic app from the repository [dummy-spring-boot](https://gitlab.com/bcfmkubilay/dummy-spring-boot.git).

git clone https://gitlab.com/bcfmkubilay/dummy-spring-boot.git

* Change your working directory to **dummy-spring-boot** and delete the **.git** directory.

```bash
cd dummy-spring-boot
rm -rf .git
```
*  Initiate the cloned repository to make it a git repository and push the local repository to your remote repository.

```bash
git init
git add .
git commit -m "first commit"
git remote add origin https://github.com/[your-git-account]/dummy-spring-boot.git
git push origin main
```
## Check the Maven Build Setup

* Prepare a script to build the docker images then Build Docker Compile image : Take the compiled code and package it in its distributable `JAR` format.

```bash
docker run --rm -v $HOME/.m2:/root/.m2 -v /home/ec2-user/springboot/:/app -w /app maven:3.6.3-openjdk-8 mvn clean package
```

* Prepare a Dockerfile

```bash
FROM openjdk:8-jre
ADD ./target/*.jar /app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
``` 

# Run docker image build

```bash
docker build -t springboot-app .
```

# Run docker image run

```bash
docker run -d -p 80:8080 springboot-app 
```

### Install kubectl

- Launch an AWS EC2 instance of Amazon Linux 2 AMI with security group allowing SSH.

- Connect to the instance with SSH.

- Update the installed packages and package cache on your instance.

```bash
$ sudo yum update -y
```

- Download the Amazon EKS vended kubectl binary.

```bash
$ curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.17.9/2020-08-04/bin/linux/amd64/kubectl
```

- Apply execute permissions to the binary.

```bash
$ chmod +x ./kubectl
```

- Move kubectl to a folder that is in your path.

```bash
$ sudo mv ./kubectl /usr/local/bin
```

- After you install kubectl , you can verify its version with the following command:

```bash
$ kubectl version --short --client
```

### Install eksctl

- Download and extract the latest release of eksctl with the following command.

```bash
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

- Move the extracted binary to /usr/local/bin.

```bash
$ sudo mv /tmp/eksctl /usr/local/bin
```

- Test that your installation was successful with the following command.

```bash
$ eksctl version
```

## Creating the Kubernetes Cluster on EKS

- If needed create ssh-key with commnad `ssh-keygen -f ~/.ssh/id_rsa`

- Configure AWS credentials.

```bash
$ aws configure
```

- Create an EKS cluster via `eksctl`. It will take a while.

```bash
$ eksctl create cluster --region us-east-1 --node-type t2.medium --nodes 1 --nodes-min 1 --nodes-max 2 --node-volume-size 8 --name mycluster
```

- Create a springboot-deployment.yaml and input text below. 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot
  labels:
    app: springboot
spec:
  replicas: 3
  selector:
    matchLabels:
      app: springboot
  template:
    metadata:
      labels:
        app: springboot
    spec:
      containers:
      - name: springboot
        image: 370639238640.dkr.ecr.us-east-1.amazonaws.com/bestcloud/spring-boot-app
        ports:
        - containerPort: 8080
```

- Create the deployment with `kubectl apply` command.

```bash
kubectl apply -f springboot-deployment.yaml
```

# Create a `springboot-service.yaml` file with following content and explain fields of it.

```yaml
apiVersion: v1
kind: Service   
metadata:
  name: springboot-svc
  labels:
    app: springboot-svc
spec:
  type: LoadBalancer 
  ports:
  - port: 8080 
    targetPort: 8080
  selector:
    app: springboot-app
``` 
```bash
kubectl apply -f springboot-service.yaml
```
```bash
kubectl get pod
```

```bash
kubectl get svc
```

### Create a ingress service.yaml

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-service
  annotations:
    kubernetes.io/ingress.class: 'nginx'
    nginx.ingress.kubernetes.io/use-regex: 'true'
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
    - http:
        paths:
          - path: /?(.*)
            backend:
              serviceName: springboot-svc
              servicePort: 8080
```


```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.35.0/deploy/static/provider/aws/deploy.yaml
```

```bash
kubectl apply -f ingress-service.yaml
```
```
kubectl get ingress
```

# Prepare a Jenkinsfile for pipeline 
```bash
pipeline {
    agent { label "master" }
    environment {
        ECR_REGISTRY = "370639238640.dkr.ecr.us-east-1.amazonaws.com"
        APP_REPO_NAME= "bestcloud/spring-boot-app"
        PATH="/usr/local/bin/:${env.PATH}"
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
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }
        stage('Deploy') {
            steps {
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker pull "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
                // sh 'docker run -dp 80:8080 "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
                sh 'kubectl apply -f .'

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

```
# Blue-green deployment file added

```bash
pipeline {
    agent { label "master" }
    environment {
        ECR_REGISTRY = "370639238640.dkr.ecr.us-east-1.amazonaws.com"
        APP_REPO_NAME= "bestcloud/spring-boot-app"
        PATH="/usr/local/bin/:${env.PATH}"
    }
    stages {
        stage('Build Docker Compile image') {
            steps {
                sh 'docker run --rm -v $HOME/.m2:/root/.m2 -v $WORKSPACE/springboot-app-v2:/app -w /app maven:3.6.3-openjdk-8 mvn clean package'
                sh 'docker image ls'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build --force-rm -t "$ECR_REGISTRY/$APP_REPO_NAME:v2" .'
                sh 'docker image ls'
            }
        }
        stage('Push Image to ECR Repo') {
            steps {
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:v2"'
            }
        }
        stage('Deploy') {
            steps {
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker pull "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
                // sh 'docker run -dp 80:8080 "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
                sh 'kubectl apply -f ./springboot-app-v2'

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
```


```bash
$ eksctl get cluster
NAME            REGION
mycluster       us-east-2
$ eksctl delete cluster mycluster

```
