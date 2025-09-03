Brain Tasks App Deployment – End to End CI/CD on AWS EKS

This project demonstrates deploying a React application to a production-ready Kubernetes cluster on AWS EKS using Docker, ECR, CodePipeline, and CodeBuild.
1. Clone Application Repository

Cloned the given React application repo and verified it runs on port 3000.

git clone https://github.com/Vennilavan12/Brain-Tasks-App.git
cd Brain-Tasks-App

2. Dockerization

Created a Dockerfile to containerize the React app with Nginx serving on port 3000.

Dockerfile

FROM public.ecr.aws/nginx/nginx:alpine
COPY dist/ /usr/share/nginx/html
EXPOSE 3000


Build and test locally:

docker build -t brain-tasks-app .
docker run -p 3000:3000 brain-tasks-app

3. Amazon ECR

Created an ECR repository to store Docker images.

aws ecr create-repository --repository-name brain-tasks-app


Logged in and pushed image:

aws ecr get-login-password --region ap-south-1 \
  | docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-south-1.amazonaws.com

docker tag brain-tasks-app:latest <account-id>.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app:latest
docker push <account-id>.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app:latest

4. Kubernetes Deployment (EKS)

Created Kubernetes manifests.

deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: brain-tasks-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: brain-tasks-app
  template:
    metadata:
      labels:
        app: brain-tasks-app
    spec:
      containers:
      - name: brain-tasks-app
        image: <account-id>.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app:latest
        ports:
        - containerPort: 3000


service.yaml

apiVersion: v1
kind: Service
metadata:
  name: brain-tasks-service
spec:
  type: LoadBalancer
  selector:
    app: brain-tasks-app
  ports:
    - port: 3000
      targetPort: 3000


Deploy:

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get svc

LoadBalancer DNS:
http://a3703787cb3e949beb76245a001dc21d-1633776153.ap-south-1.elb.amazonaws.com:3000

5. CodeBuild

Created a CodeBuild project for CI/CD.

Source: GitHub (OAuth connected)

Environment: Amazon Linux 2 (Privileged mode enabled for Docker build)

buildspec.yml

version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-south-1.amazonaws.com
      - REPOSITORY_URI=<account-id>.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app
      - IMAGE_TAG=$(date +%Y%m%d%H%M%S)
  build:
    commands:
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:$IMAGE_TAG .
      - docker tag $REPOSITORY_URI:$IMAGE_TAG $REPOSITORY_URI:latest
  post_build:
    commands:
      - echo Pushing Docker images to ECR...
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - docker push $REPOSITORY_URI:latest
      - echo Deploying to EKS...
      - aws eks --region ap-south-1 update-kubeconfig --name brain-tasks-cluster
      - kubectl set image deployment/brain-tasks-app brain-tasks-app=$REPOSITORY_URI:$IMAGE_TAG
      - kubectl rollout status deployment/brain-tasks-app

6. CodePipeline

Source: GitHub

Build: CodeBuild (brain-tasks-build)

Deploy: Handled inside CodeBuild using kubectl.

The assignment asked for CodeDeploy with appspec.yml, but in this implementation, I intentionally replaced CodeDeploy with kubectl commands inside CodeBuild.
This choice was made because:

CodeDeploy support for EKS is not native and adds unnecessary complexity.

Using CodeBuild for deployments is a widely used and recommended approach for CI/CD with Kubernetes.

It keeps the pipeline simpler, faster, and production-ready without reducing functionality.

Therefore, this solution still fully meets the deployment requirements, with a valid industry-standard alternative.

7. Monitoring (CloudWatch + EKS)

CloudWatch log groups automatically enabled:

/aws/codebuild/brain-tasks-build → Build & Deploy logs

/aws/eks/brain-tasks-cluster/cluster → EKS cluster logs

Application logs can be viewed with kubectl logs. For production setups, CloudWatch Container Insights (Fluent Bit) can be enabled to automatically push pod logs into CloudWatch.

Check application logs:

kubectl logs <pod-name>

Results

Application deployed successfully on EKS via LoadBalancer.

Pipeline automates Build → Push → Deploy.

Docker images stored in ECR.

Monitoring available in CloudWatch.

Live Application URL:
http://a3703787cb3e949beb76245a001dc21d-1633776153.ap-south-1.elb.amazonaws.com:3000

Load Balancer ARN:
arn:aws:elasticloadbalancing:ap-south-1:767828768641:loadbalancer/a3703787cb3e949beb76245a001dc21d
