# Deploy FTGO on Amazon EKS

This guide deploys this FTGO microservice project to Amazon EKS using the Kubernetes manifests already in this repository.

The project contains Kubernetes YAML in:

- `deployment/kubernetes/stateful-services/`
- `deployment/kubernetes/cdc-service/`
- `ftgo-*/src/deployment/kubernetes/`

The repo also contains an old deploy script at `deployment/kubernetes/scripts/kubernetes-deploy-all.sh`, but it references `deployment/kubernetes/cdc-services/`. In this checkout the folder is `deployment/kubernetes/cdc-service/`, so the commands below deploy the manifests manually.

## 1. Prepare a Fresh WSL Ubuntu Environment

This section assumes you just installed a fresh WSL Ubuntu distribution and only have a terminal.

Update Ubuntu and install basic utilities:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl unzip git tar gzip ca-certificates gnupg lsb-release
```

Do not run this command:

```bash
sudo apt install kubectl docker aws eksctl
```

Those package names are not available from the default Ubuntu repositories. Install the tools with the commands in the next section instead.

If your project is already on Windows, you can work from the mounted Windows path:

```bash
cd "/mnt/c/Users/NeaRzro/Documents/DOKUMEN-KULIAH/Semester 6/Cloud/Microservice/ftgo-application"
```

If you are starting from an empty WSL environment, clone the repository instead:

```bash
git clone <your-repository-url>
cd ftgo-application
```

For Docker, the recommended setup on Windows + WSL is Docker Desktop:

1. Install Docker Desktop on Windows.
2. Open Docker Desktop.
3. Go to Settings > Resources > WSL Integration.
4. Enable integration for your Ubuntu distro.
5. Restart the WSL terminal.

Verify Docker from WSL:

```bash
docker version
docker compose version
```

If Docker is not found, make sure Docker Desktop is running and WSL integration is enabled.

## 2. Install Required Tools

Install these tools on your local machine or WSL environment:

- AWS CLI
- eksctl
- kubectl
- Docker

Use the official install pages, or install with these common methods:

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# eksctl
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/

# AWS CLI v2
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
cd -
```

Verify the tools:

```bash
kubectl version --client
aws --version
eksctl version
```

## 3. Configure AWS

Choose a region. The examples use `ap-southeast-1`; change it if needed.

```bash
aws configure

export AWS_REGION=ap-southeast-1
export CLUSTER_NAME=ftgo-test-cluster
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

If you open a new WSL terminal, set these variables again before continuing. Verify them:

```bash
echo $AWS_REGION
echo $CLUSTER_NAME
echo $AWS_ACCOUNT_ID
echo $ECR_REGISTRY
```

`AWS_REGION` must not be empty. If it is empty, AWS CLI commands can fail with an invalid endpoint such as:

```text
https://api.ecr..amazonaws.com
```

Confirm identity:

```bash
aws sts get-caller-identity
```

## 4. Create an EKS Cluster

For a demo deployment, use at least two worker nodes. FTGO runs many pods, so avoid very small nodes.

```bash
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --nodes 2 \
  --node-type t3.large \
  --managed
```

Configure `kubectl`:

```bash
aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION
kubectl get nodes
```

## 5. Build the Application JARs

This project needs the Spring Boot JARs before Docker images can be built. If Java is not installed locally, build with Docker:

```bash
docker run --rm -v "$PWD:/workspace" -w /workspace gradle:6.9.4-jdk8 gradle assemble -x test
```

Expected result:

```text
BUILD SUCCESSFUL
```

## 6. Create ECR Repositories

Create one ECR repository for each custom image:

```bash
for repo in \
  ftgo-consumer-service \
  ftgo-order-service \
  ftgo-kitchen-service \
  ftgo-restaurant-service \
  ftgo-accounting-service \
  ftgo-order-history-service \
  ftgo-api-gateway \
  dynamodblocal-init \
  mysql
do
  aws ecr describe-repositories --repository-names "$repo" --region "$AWS_REGION" >/dev/null 2>&1 \
    || aws ecr create-repository --repository-name "$repo" --region "$AWS_REGION"
done
```

Login Docker to ECR:

```bash
aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin "$ECR_REGISTRY"
```

## 7. Build and Push Docker Images

Load the same image version values used by the local Docker Compose setup:

```bash
export EVENTUATE_COMMON_VERSION=0.15.0.RELEASE
export EVENTUATE_CDC_VERSION=0.13.0.RELEASE
export EVENTUATE_SAGA_VERSION=0.19.0.RELEASE
export EVENTUATE_JAVA_BASE_IMAGE_VERSION=BUILD-15
export EVENTUATE_MESSAGING_KAFKA_IMAGE_VERSION=0.15.0.RELEASE
```

Build the service images:

```bash
docker build --build-arg baseImageVersion=$EVENTUATE_JAVA_BASE_IMAGE_VERSION -t ftgo-consumer-service:latest ./ftgo-consumer-service
docker build --build-arg baseImageVersion=$EVENTUATE_JAVA_BASE_IMAGE_VERSION -t ftgo-order-service:latest ./ftgo-order-service
docker build --build-arg baseImageVersion=$EVENTUATE_JAVA_BASE_IMAGE_VERSION -t ftgo-kitchen-service:latest ./ftgo-kitchen-service
docker build --build-arg baseImageVersion=$EVENTUATE_JAVA_BASE_IMAGE_VERSION -t ftgo-restaurant-service:latest ./ftgo-restaurant-service
docker build --build-arg baseImageVersion=$EVENTUATE_JAVA_BASE_IMAGE_VERSION -t ftgo-accounting-service:latest ./ftgo-accounting-service
docker build --build-arg baseImageVersion=$EVENTUATE_JAVA_BASE_IMAGE_VERSION -t ftgo-order-history-service:latest ./ftgo-order-history-service
docker build --build-arg baseImageVersion=$EVENTUATE_JAVA_BASE_IMAGE_VERSION -t ftgo-api-gateway:latest ./ftgo-api-gateway
docker build --build-arg EVENTUATE_COMMON_VERSION=$EVENTUATE_COMMON_VERSION --build-arg EVENTUATE_SAGA_VERSION=$EVENTUATE_SAGA_VERSION -t mysql:latest ./mysql
docker build -t dynamodblocal-init:latest ./dynamodblocal-init
```

Tag and push them to ECR:

```bash
for image in \
  ftgo-consumer-service \
  ftgo-order-service \
  ftgo-kitchen-service \
  ftgo-restaurant-service \
  ftgo-accounting-service \
  ftgo-order-history-service \
  ftgo-api-gateway \
  dynamodblocal-init \
  mysql
do
  docker tag "$image:latest" "$ECR_REGISTRY/$image:latest"
  docker push "$ECR_REGISTRY/$image:latest"
done
```

## 8. Update Kubernetes Image Names

The Kubernetes manifests currently use public image names such as:

```yaml
image: msapatterns/ftgo-api-gateway:latest
```

Replace those with your ECR image names:

```yaml
image: <account-id>.dkr.ecr.<region>.amazonaws.com/ftgo-api-gateway:latest
```

Update these files:

- `deployment/kubernetes/stateful-services/ftgo-mysql-deployment.yml`
- `deployment/kubernetes/stateful-services/ftgo-dynamodb-local.yml`
- `ftgo-accounting-service/src/deployment/kubernetes/ftgo-accounting-service.yml`
- `ftgo-api-gateway/src/deployment/kubernetes/ftgo-api-gateway.yml`
- `ftgo-consumer-service/src/deployment/kubernetes/ftgo-consumer-service.yml`
- `ftgo-kitchen-service/src/deployment/kubernetes/ftgo-kitchen-service.yml`
- `ftgo-order-history-service/src/deployment/kubernetes/ftgo-order-history-service.yml`
- `ftgo-order-service/src/deployment/kubernetes/ftgo-order-service.yml`
- `ftgo-restaurant-service/src/deployment/kubernetes/ftgo-restaurant-service.yml`

You can replace them with `sed`:

```bash
grep -R "msapatterns/" -n deployment/kubernetes ftgo-*/src/deployment/kubernetes
grep -R "image:" -n deployment/kubernetes ftgo-*/src/deployment/kubernetes
```

Then edit each matching file and use:

```text
$ECR_REGISTRY/<image-name>:latest
```

Do not replace third-party images unless you intentionally mirror them:

- `confluentinc/cp-kafka:5.2.4`
- `confluentinc/cp-zookeeper:5.2.4`
- `cnadiminti/dynamodb-local:2017-04-22_beta`
- `eventuateio/eventuate-cdc-service:0.4.0.RELEASE`

## 9. Deploy Stateful Services

Deploy MySQL, Kafka, Zookeeper, DynamoDB Local, and secrets:

```bash
kubectl apply -f deployment/kubernetes/stateful-services/
```

Wait until these pods are ready:

```bash
kubectl get pods -w
```

You should see pods similar to:

```text
ftgo-mysql-0
ftgo-kafka-0
ftgo-zookeeper-0
ftgo-dynamodb-local-0
```

Check details if one is stuck:

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

## 10. Deploy CDC Service

Deploy the CDC service:

```bash
kubectl apply -f deployment/kubernetes/cdc-service/
```

Check it:

```bash
kubectl get pods
kubectl logs <cdc-pod-name>
```

## 11. Deploy FTGO Microservices

Deploy all application services:

```bash
kubectl apply -f ftgo-accounting-service/src/deployment/kubernetes/
kubectl apply -f ftgo-api-gateway/src/deployment/kubernetes/
kubectl apply -f ftgo-consumer-service/src/deployment/kubernetes/
kubectl apply -f ftgo-kitchen-service/src/deployment/kubernetes/
kubectl apply -f ftgo-order-history-service/src/deployment/kubernetes/
kubectl apply -f ftgo-order-service/src/deployment/kubernetes/
kubectl apply -f ftgo-restaurant-service/src/deployment/kubernetes/
```

Verify:

```bash
kubectl get pods
kubectl get svc
```

If a service crashes while MySQL is still starting, wait until MySQL is ready and restart the deployment:

```bash
kubectl rollout restart deployment/ftgo-consumer-service
kubectl rollout restart deployment/ftgo-order-service
kubectl rollout restart deployment/ftgo-kitchen-service
kubectl rollout restart deployment/ftgo-restaurant-service
kubectl rollout restart deployment/ftgo-accounting-service
kubectl rollout restart deployment/ftgo-delivery-service
```

Only run the `ftgo-delivery-service` restart if you have a Kubernetes manifest for it. This checkout has a Docker Compose service for delivery, but no `ftgo-delivery-service/src/deployment/kubernetes/` manifest.

## 12. Access the Application

The simplest way to access the services is port forwarding:

```bash
kubectl port-forward svc/ftgo-api-gateway 8087:8080
```

Then open:

```text
http://localhost:8087/actuator/health
```

For Swagger UIs:

```bash
kubectl port-forward svc/ftgo-consumer-service 8081:8080
kubectl port-forward svc/ftgo-order-service 8082:8080
kubectl port-forward svc/ftgo-restaurant-service 8084:8080
kubectl port-forward svc/ftgo-order-history-service 8086:8080
```

Open:

```text
http://localhost:8081/swagger-ui/index.html
http://localhost:8082/swagger-ui/index.html
http://localhost:8084/swagger-ui/index.html
http://localhost:8086/swagger-ui/index.html
```

The root URL `/` may show a Spring Whitelabel 404 page. That is expected because this project is an API demo, not a web homepage.

## 13. Optional: Expose API Gateway Publicly

For a simple demo, change the `ftgo-api-gateway` Service from `ClusterIP` to `LoadBalancer`.

Edit:

```text
ftgo-api-gateway/src/deployment/kubernetes/ftgo-api-gateway.yml
```

Set the service type:

```yaml
spec:
  type: LoadBalancer
```

Apply again:

```bash
kubectl apply -f ftgo-api-gateway/src/deployment/kubernetes/
kubectl get svc ftgo-api-gateway
```

Wait for an external hostname:

```bash
kubectl get svc ftgo-api-gateway -w
```

Then open:

```text
http://<external-load-balancer-hostname>/actuator/health
```

For production, use an AWS Load Balancer Controller and an Ingress instead of exposing many services directly.

## 14. Troubleshooting

Check pods:

```bash
kubectl get pods
```

Check services:

```bash
kubectl get svc
```

Check logs:

```bash
kubectl logs <pod-name>
```

Follow logs:

```bash
kubectl logs -f <pod-name>
```

Describe a failing pod:

```bash
kubectl describe pod <pod-name>
```

Common issues:

- `ImagePullBackOff`: the ECR image URL is wrong, the image was not pushed, or the EKS nodes cannot pull from ECR.
- `CrashLoopBackOff` with MySQL connection errors: wait until `ftgo-mysql-0` is ready, then restart the app deployment.
- `ErrImagePull`: check the image tag and region.
- `Pending`: your cluster does not have enough CPU or memory.
- Whitelabel 404 at `/`: expected; use Swagger or `/actuator/health`.

### eksctl exceeded max wait time

If cluster creation fails with:

```text
exceeded max wait time for StackCreateComplete waiter
failed to create cluster "ftgo-test-cluster"
```

Do not immediately run `eksctl create cluster` again. First inspect the CloudFormation stack events:

```bash
aws cloudformation describe-stack-events \
  --region $AWS_REGION \
  --stack-name eksctl-ftgo-test-cluster-cluster \
  --query "StackEvents[?ResourceStatus=='CREATE_FAILED' || ResourceStatus=='ROLLBACK_IN_PROGRESS' || ResourceStatus=='ROLLBACK_COMPLETE'].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]" \
  --output table
```

Also inspect the nodegroup stack mentioned in the error. Replace the stack name with the exact value from your terminal:

```bash
aws cloudformation describe-stack-events \
  --region $AWS_REGION \
  --stack-name eksctl-ftgo-test-cluster-nodegroup-ng-ce072664 \
  --query "StackEvents[?ResourceStatus=='CREATE_FAILED' || ResourceStatus=='ROLLBACK_IN_PROGRESS' || ResourceStatus=='ROLLBACK_COMPLETE'].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]" \
  --output table
```

Common causes:

- Your AWS account does not have enough EC2 quota for the selected node type.
- The selected region has limited capacity for `t3.large`.
- Required IAM permissions are missing.
- VPC, subnet, NAT gateway, or internet gateway resources failed to create.
- The nodegroup failed to join the cluster.

After checking the reason, clean up the partial cluster before retrying:

```bash
eksctl delete cluster --region=$AWS_REGION --name=$CLUSTER_NAME
```

Then retry with a smaller or different node type if the issue was EC2 capacity or quota:

```bash
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --nodes 2 \
  --node-type t3.medium \
  --managed
```

For this project, `t3.large` is more comfortable. `t3.medium` may work for testing, but pods can become `Pending` if the cluster runs out of memory.

## 15. Cleanup

Delete the application services:

```bash
kubectl delete -f ftgo-accounting-service/src/deployment/kubernetes/
kubectl delete -f ftgo-api-gateway/src/deployment/kubernetes/
kubectl delete -f ftgo-consumer-service/src/deployment/kubernetes/
kubectl delete -f ftgo-kitchen-service/src/deployment/kubernetes/
kubectl delete -f ftgo-order-history-service/src/deployment/kubernetes/
kubectl delete -f ftgo-order-service/src/deployment/kubernetes/
kubectl delete -f ftgo-restaurant-service/src/deployment/kubernetes/
```

Delete CDC and stateful services:

```bash
kubectl delete -f deployment/kubernetes/cdc-service/
kubectl delete -f deployment/kubernetes/stateful-services/
```

Delete the EKS cluster:

```bash
eksctl delete cluster --name $CLUSTER_NAME --region $AWS_REGION
```

Delete ECR repositories if you no longer need the images:

```bash
for repo in \
  ftgo-consumer-service \
  ftgo-order-service \
  ftgo-kitchen-service \
  ftgo-restaurant-service \
  ftgo-accounting-service \
  ftgo-order-history-service \
  ftgo-api-gateway \
  dynamodblocal-init \
  mysql
do
  aws ecr delete-repository --repository-name "$repo" --region "$AWS_REGION" --force
done
```
