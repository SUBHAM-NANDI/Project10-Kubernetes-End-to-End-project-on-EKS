# Project10-Kubernetes-End-to-End-project-on-EKS

### Step 1: Prerequisites
1. **AWS Account**: Ensure you have an AWS account set up.
2. **EKS Cluster**: Create an EKS cluster (if not already created).
3. **kubectl**: Install and configure `kubectl` to interact with your EKS cluster.
4. **eksctl**: Install `eksctl`, a command-line tool for creating and managing EKS clusters.
5. **AWS CLI**: Install the AWS CLI and configure it with your AWS credentials.
6. **IAM Permissions**: Ensure your IAM user has the necessary permissions to create resources like EKS, IAM roles, and policies.

### Step 2: Create the EKS Cluster
Use `eksctl` to create an EKS cluster:
```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

### Step 3: Configure kubectl
Update your `kubectl` config to interact with your new EKS cluster:
```bash
aws eks --region us-east-1 update-kubeconfig --name demo-cluster
```

### Step 4: Create a Fargate Profile
Create a Fargate profile to define how pods will run:
```bash
eksctl create fargateprofile --cluster demo-cluster --name fargate-profile --namespace game2048
```

### Step 5: Download the kubeconfig File
To use `kubectl`, download the kubeconfig file:
```bash
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

### Step 6: Create the Namespace
Create a Kubernetes namespace where your application will run:
```bash
kubectl create namespace game2048
```

### Step 7: Deploy the Application
1. **Create a Deployment and Service**:
   Create a YAML file (e.g., `2048-app.yaml`) with the following content:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: game2048
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2048
  namespace: game2048
spec:
  replicas: 4
  selector:
    matchLabels:
      app: app2048
  template:
    metadata:
      labels:
        app: app2048
    spec:
      containers:
      - name: app2048
        image: your-docker-image:latest  # Replace with your image
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: app2048
  namespace: game2048
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: app2048
```
Deploy the application:
```bash
kubectl apply -f 2048-app.yaml
```

### Step 8: Set Up Ingress Controller
1. **Install the OIDC Connector**:
   Run the following command to associate the IAM OIDC provider with your EKS cluster:
```bash
eksctl utils associate-iam-oidc-provider --region us-east-1 --cluster demo-cluster --approve
```

2. **Create IAM Policy for ALB**:
   Save the following policy as a JSON file (e.g., `alb-policy.json`):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:*",
        "ec2:*",
        "cloudwatch:*",
        "iam:PassRole",
        "logs:*"
      ],
      "Resource": "*"
    }
  ]
}
```
Create the IAM policy:
```bash
aws iam create-policy --policy-name ALBControllerIAMPolicy --policy-document file://alb-policy.json
```

3. **Create an IAM Role for ALB Controller**:
```bash
eksctl create iamserviceaccount \
  --region us-east-1 \
  --name alb-ingress-controller \
  --namespace kube-system \
  --cluster demo-cluster \
  --attach-policy-arn arn:aws:iam::<Your_AWS_Account_ID>:policy/ALBControllerIAMPolicy \
  --approve
```

4. **Deploy the ALB Ingress Controller**:
```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-load-balancer-controller//deploy/overlays/helm-controller?ref=release-2.4"
```

### Step 9: Create the Ingress Resource
Create an Ingress resource YAML file (e.g., `ingress.yaml`):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app2048-ingress
  namespace: game2048
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app2048
                port:
                  number: 80
```
Deploy the Ingress resource:
```bash
kubectl apply -f ingress.yaml
```

### Step 10: Access the Application
1. **Get the Ingress URL**:
   After deploying, check the Ingress resource to get the ALB URL:
```bash
kubectl get ingress -n game2048
```
2. **Access the Application**:
   Open a browser and navigate to the ALB URL to access your application.

### Step 11: Cleanup
When done testing, you can delete the resources created:
```bash
eksctl delete cluster --name demo-cluster --region us-east-1
```
