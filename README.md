
---

# Deploying a 2048 Game on AWS EKS

## Install Necessary Tools

1. **Install `kubectl`** - the command-line tool to interact with Kubernetes clusters.
2. **Install `eksctl`** - a command-line utility specifically for managing EKS clusters.
3. **Install and configure the AWS CLI**.

## Step 1: Create an EKS Cluster

Use the following command to create an EKS cluster using `eksctl`:

```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

Replace `"demo-cluster"` with your desired cluster name and `"us-east-1"` with your desired region. This command creates a cluster with a managed control plane and Fargate worker nodes. It also automatically sets up networking components like public and private subnets.

> **Important**: This process can take 15-20 minutes or longer depending on your network speed.

After the cluster is created, update your local `kubeconfig` file to interact with it using `kubectl`:

```bash
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

This allows you to manage the cluster from your command line.

## Step 2: Create a Fargate Profile

EKS requires you to create a Fargate profile to specify which namespaces your Fargate pods can be deployed to.

```bash
eksctl create fargateprofile \
--cluster demo-cluster \
--region us-east-1 \
--name alb-sample-app \
--namespace game2048
```

Replace `"demo-cluster"` with your cluster name, `"us-east-1"` with your region, `"alb-sample-app"` with your desired profile name, and `"game2048"` with your chosen namespace.

## Step 3: Create Pod Deployment, Service, and Ingress

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

## Step 4: Create IAM Resources for the ALB Ingress Controller

### Create IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```

The ALB Ingress Controller needs appropriate IAM permissions to interact with AWS resources like Application Load Balancers (ALBs).

### Create IAM Policy

Download the IAM policy JSON:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

Create the IAM policy:

```bash
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam_policy.json
```

### Create IAM Service Account

> **Important**: Don't forget to add your cluster name and account ID before creating the role.

```bash
eksctl create iamserviceaccount \
--cluster=<your-cluster-name> \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--role-name AmazonEKSLoadBalancerControllerRole \
--attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve
```

This allows you to use IAM roles for authentication and authorization within your Kubernetes cluster.

## Step 5: Install the ALB Ingress Controller using Helm

1. **Add the Helm repository for EKS**:

   ```bash
   helm repo add eks https://aws.github.io/eks-charts
   ```

2. **Update the Helm repository**:

   ```bash
   helm repo update eks
   ```

3. **Install the ALB Ingress Controller**:

   Deploy the ALB Ingress Controller to your cluster using the official Helm chart. Modify the following command with your specific VPC ID, region, and cluster name. Ensure that the service account specified in the Helm chart matches the one you created earlier.

   ```bash
   helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
   -n kube-system \
   --set clusterName=<your-cluster-name> \
   --set serviceAccount.create=false \
   --set serviceAccount.name=aws-load-balancer-controller \
   --set region=<region> \
   --set vpcId=<your-vpc-id>
   ```

## Step 6: Verify Deployment

Check if the AWS Load Balancer Controller deployments are running. Ensure two AWS Load Balancer Controller replicas are created:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

## Step 7: Access the 2048 Game

Retrieve the address of the ALB to access the game:

```bash
kubectl get ingress -n game2048
```

Copy the address and go to your browser's URL to access the game:

```
http://<alb-address>
```

## Step 8: Clean Up

After completing the project, delete the cluster to avoid incurring additional charges:

```bash
eksctl delete cluster --name demo-cluster --region us-east-1
```

Congratulations! You successfully deployed the 2048 game on AWS EKS!

--- 


