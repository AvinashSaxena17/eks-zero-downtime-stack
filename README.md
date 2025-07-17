## Project Overview :

This project deploys a production-ready cloud-native application on AWS EKS, featuring auto-scaling with HPA and cluster auto scaler, WAF-based L4/L7 rate limiting via ALB/NLB, secret encryption using  with AWS KMS, canary deployments using Argo Rollouts.

## 📦 Project Description:

This project sets up a production-grade, cloud-native application infrastructure on AWS using Amazon EKS.
It features dynamic resource scaling with Horizontal Pod Autoscaler (HPA) and Cluster Autoscaler, robust L4/L7 rate limiting using AWS WAF integrated with ALB/NLB, and secure secret management using AWS Secrets Manager, where sensitive data like database credentials are securely stored and accessed at runtime via IAM Roles for Service Accounts (IRSA).
Canary deployments are enabled using Argo Rollouts to ensure zero downtime and safer releases.

## 📌 Prerequisites

- AWS CLI configured

  ---


 **Step-by-Step Setup**

1. EKS Cluster Setup

2. Install AWS Load Balancer Controller

3. Deploying an Amazon ECR Image with Kubernetes Using ALB Ingress Controller (All-in-One Manifest)

4. Configure HPA (Horizontal Pod Autoscaler)
 
5. Install Cluster Autoscaler

6. Configure AWS WAF

7. Zero Downtime Deployments with Argo Rollouts

8. Test the load.

9. Secure Database Credentials Using AWS Secrets Manager in EKS
   

- Conclusion

- Key Achievements and Technical SKills




## ✅ Step 1: Create EKS Cluster

```bash
eksctl create cluster \
  --name prod-cluster \
  --region ap-south-1 \
  --nodegroup-name prod-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 6 \
  --managed
```




![App Screenshot](https://github.com/AvinashSaxena17/eks-zero-downtime-stack/blob/main/screenshots/creating%20cluster%20via%20CLI.png)

**Verify cluster:**
```bash
kubectl get nodes
```
![App Screenshot](https://github.com/AvinashSaxena17/eks-zero-downtime-stack/blob/main/screenshots/Get%20nodes.png)


## ✅ Here's what eksctl creates automatically behind the scenes:
🔧 1. VPC
A new VPC with:

2 or more public subnets (in different Availability Zones)

2 or more private subnets (for managed nodegroups)

🔧 2. Subnets
Automatically created based on your region (e.g., ap-south-1) and AZs.

Labeled as private/public depending on whether nodes need public IPs.

🔧 3. Internet Gateway / NAT Gateway
If private subnets are created, eksctl adds a NAT Gateway for outbound internet access.

Internet Gateway for public subnets.

🔧 4. Route Tables
Routes between subnets and gateways are auto-configured.

🔧 5. Security Groups
For both control plane and node group communication.

🔧 6. IAM Roles
Control plane and node group IAM roles and policies are created.

Includes policies for EKS, EC2, autoscaling, etc.

🔧 7. EKS Control Plane
Your EKS cluster control plane is provisioned and connected to the above infra.

🔧 8. Managed Node Group
Your --managed flag tells eksctl to create a managed node group with 2 t3.medium EC2 instances.

## 📌 Result:
After running your command, you get a production-ready EKS cluster with fully working networking — all auto-managed unless overridden.


## ✅ Step 2 Install AWS Load Balancer Controller :

## Associate IAM OIDC Provider with EKS Cluster
```
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster prod-cluster \
  --approve
```
## a) Create IAM Role using eksctl:

**Download an IAM policy for the AWS Load Balancer Controller that allows it to make calls to AWS APIs on your behalf.**
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json
```

**Create an IAM policy using the policy downloaded in the previous step.**
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

***Replace the values for cluster name, region code, and account ID.**
```
eksctl create iamserviceaccount \
    --cluster=prod-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region ap-south-1 \
    --approve
```
 

## b) Install AWS Load Balancer Controller:

**Add the eks-charts Helm chart repository**
```
helm repo add eks https://aws.github.io/eks-charts
```

**Update your local repo to make sure that you have the most recent charts.**
```
helm repo update eks
```

**Install the AWS Load Balancer Controller.**
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=prod-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.13.0
```

![App Screenshot](https://github.com/AvinashSaxena17/eks-zero-downtime-stack/blob/main/screenshots/Load%20balancer%20controller%20(HELM).png)


for more info click on : [documentation](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)


**Verify It’s Running**
```
kubectl get pods -n kube-system
```

## ✅ Step 3: Deploying an Amazon ECR Image with Kubernetes Using ALB Ingress Controller (All-in-One Manifest)

**apply YAML manifest that includes Deployment, Service, and Ingress resources to deploy a Docker image from Amazon ECR using AWS ALB Ingress Controller:**


```
kubectl apply -f my-app-alb.yaml
```

**🌐 Get Ingress URL**
```
kubectl get ingress my-app-ingress
```

- "Access your application using the Ingress URL provided by the ALB to verify it’s running successfully".


![App Screenshot](https://github.com/AvinashSaxena17/eks-zero-downtime-stack/blob/main/screenshots/application%20deployed.png)


## ✅ Step 4: Configure HPA (Horizontal Pod Autoscaler):

**a) Install metrics-server first.**

- Metrics Server collects pod CPU/memory usage so that HPA can make decisions.

  **Apply:**
```
 kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

- Wait a minute, then confirm it's working:

```
kubectl get deployment metrics-server -n kube-system

```
![App Screenshot](https://github.com/AvinashSaxena17/eks-zero-downtime-stack/blob/main/screenshots/check%20metrics%20server%20pod.png)

**Then define HPA YAMLs to scale pods based on CPU/memory/custom metrics**

**Apply:**

```
kubectl apply -f hpa.yaml
```

**Verify HPA is working:**

```
kubectl get hpa
```

![App Screenshot](https://github.com/AvinashSaxena17/eks-zero-downtime-stack/blob/main/screenshots/get%20hpa-1.png)

- HPA automatically scales the number of pods in a Kubernetes Deployment or ReplicaSet based on CPU, memory, or custom metrics.

**🔑 Key Points:**

- Ensures app performance during high load.

- Saves resources during low usage.

- Requires metrics-server to monitor resource usage.


## ✅ Step 5: Install Cluster Autoscaler:

- Implemented Cluster Autoscaler in EKS to automatically scale worker nodes based on workload demand, improving application reliability and reducing cloud costs by removing unused worker nodes or EC2 instances.


**🔐 1. Get your Node Group’s Auto Scaling Group name:**
```
aws autoscaling describe-auto-scaling-groups \
  --region ap-south-1 \
  --query 'AutoScalingGroups[*].AutoScalingGroupName' \
  --output text
```

- Copy the correct ASG name associated with your EKS nodes.


**Tag the ASG for Cluster Autoscaler**
```
aws autoscaling create-or-update-tags \
  --tags "ResourceId=<YOUR_ASG_NAME>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=true" \
         "ResourceId=<YOUR_ASG_NAME>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/prod-cluster,Value=owned,PropagateAtLaunch=true" \
  --region ap-south-1
```

**Create IAM Policy for Cluster Autoscaler**

**Download the policy JSON :**

```
curl -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-policy.json

```

**Then create the policy:**
```
aws iam create-policy \
  --policy-name ClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json
```

**Create IAM Role & ServiceAccount for Cluster Autoscaler**

```
eksctl create iamserviceaccount \
  --cluster prod-cluster \
  --namespace kube-system \
  --name cluster-autoscaler \
  --attach-policy-arn arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:policy/ClusterAutoscalerPolicy \
  --approve
```

**Apply the cluster yaml manifests:**
```
kubectl apply -f cluster-autoscaler.yaml
```

**✅ Next Steps to Confirm It Works**

```
kubectl get pods -n kube-system | grep cluster-autoscaler
```

- Copy the pod name, then:

```
kubectl logs -n kube-system <pod-name>
```


## ✅ Step 6:  Configure AWS WAF:

- Configured AWS WAF with ALB to protect applications from common web threats and unauthorized access, enhancing security and preventing malicious traffic.

**Steps:**

- Navigate to AWS Console → WAF → Web ACL → Create
- Add Rate-based rule
    - Limit: 1000 requests per 5 minutes per IP
- Attach the Web ACL to your ALB



![App Screenshot](https://github.com/AvinashSaxena17/eks-zero-downtime-stack/blob/main/screenshots/WAF.png)


## ✅ Step 7:  Zero Downtime Deployments with Argo Rollouts:


- Integrated Argo Rollouts for automated canary deployments, reducing manual intervention and minimizing downtime by up to 90%.
  

**Install Argo Rollouts**

```
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f argo.yaml
```

## ✅ Step 8: Load Testing (See Autoscaling in Action):

**Start a load generator:**

```
kubectl run -i --tty load-generator --image=busybox /bin/sh
```

**Inside container:**
```
sh

while true; do wget -q -O- http://my-app; done
```

**Now in a separate terminal, test the load:**

**before load**

![App Screenshot](https://github.com/AvinashSaxena17/eks-zero-downtime-stack/blob/main/screenshots/get%20hpa-1.png)


**After load**


![App Screenshot](https://github.com/AvinashSaxena17/eks-zero-downtime-stack/blob/main/screenshots/get%20hpa%20after%20load.png)




## ✅ Step 9: Secure Database Credentials Using AWS Secrets Manager in EKS:

- This section explains how to store sensitive credentials (e.g., database username and password) in AWS Secrets Manager and access them securely in your Kubernetes backend application running on EKS.

**Create a Secret in AWS Secrets Manager**

```
aws secretsmanager create-secret \
  --name myapp/db-creds \
  --secret-string '{"username":"admin","password":"securepass"}'
```

**Create IAM Policy to Access the Secret**

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:ap-south-1:<account-id>:secret:myapp/db-creds-*"
    }
  ]
}

```

**Attach this policy to an IAM role.**

```
eksctl create iamserviceaccount \
  --name secret-sa \
  --namespace default \
  --cluster <cluster-name> \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/my-secret-access \
  --approve

```

**fetch this secret in backend deployment for login in databases with username and password


**In your backend application's deployment manifest, reference the ServiceAccount you created:**



## 📊 Optional: Monitor Your EKS Cluster with AWS CloudWatch Container Insights

- Amazon CloudWatch offers native monitoring for Amazon EKS via Container Insights, allowing you to track CPU, memory, disk, network, and pod-level metrics without setting up Prometheus or Grafana.


## Conclusion:

- Spearheaded deployment of a highly available, scalable, and secure cloud-native application on AWS EKS, optimizing resource utilization and ensuring zero-downtime deployments.

**Key Achievements & Technical Skills**

- AWS EKS Cluster Setup: Provisioned and managed production-grade Kubernetes clusters on AWS using eksctl, establishing robust VPC, subnet, and IAM infrastructure.
- Automated Scaling: Implemented Horizontal Pod Autoscaler (HPA) for dynamic application scaling based on resource utilization and Kubernetes Cluster Autoscaler for efficient EC2 node management, reducing operational overhead.
- Advanced Security: Configured AWS WAF integrated with ALB Ingress to enforce L4/L7 rate limiting and protect against common web exploits, enhancing application security posture.
- Zero-Downtime Deployments: Leveraged Argo Rollouts to orchestrate blue/green deployment strategies, ensuring seamless application updates with minimized user impact and rapid rollback capabilities.

  














