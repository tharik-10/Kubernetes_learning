🏗️ Phase 1: Cluster & CLI Setup
First, prepare your management machine (Ubuntu EC2) and create the EKS cluster.

1. Install Required Tools
Bash

# Install AWS CLI, Kubectl, and Eksctl
sudo apt update && sudo apt install -y docker.io unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/eksctl

Option A: Use an IAM Role (Recommended)
This is the most secure way. You "give" the EC2 instance a permanent identity without using secret keys.

Go to the AWS Console: Navigate to IAM > Roles > Create role.

Select Trusted Entity: Choose AWS Service and then EC2.

Add Permissions: Attach the AdministratorAccess policy (needed for eksctl to build VPCs, IAM roles, and clusters).

Name it: Call it SpinnakerManagerRole.

Attach to Instance: * Go to EC2 Instances in the console.

Right-click your instance (ip-172-31-16-238).

Select Actions > Security > Modify IAM role.

Choose SpinnakerManagerRole and save.

Wait about 30 seconds, then run your eksctl command again. It will work now because it will fetch credentials from the "IMDS" (Instance Metadata Service) automatically.

Option B: Use Access Keys (Quick Fix)
If you don't want to mess with IAM roles, you can manually log in with an IAM User's keys.

Run the config command:

Bash

aws configure
Enter your details:

AWS Access Key ID: [Your Key]

AWS Secret Access Key: [Your Secret]

Default region name: us-east-1

Default output format: json

Verify it works:

Bash

aws sts get-caller-identity
If this returns your Account ID and User ARN, you are ready to run the eksctl create cluster command.

2. Create the EKS Cluster
Spinnaker requires significant resources. Use at least t3.xlarge nodes.

Bash

eksctl create cluster \
  --name spin-cluster \
  --region us-east-1 \
  --nodegroup-name standard-nodes \
  --node-type t3.xlarge \
  --nodes 3 \
  --with-oidc \
  --managed

⚖️ Phase 2: AWS Load Balancer Controller Setup
This controller allows Kubernetes to "talk" to AWS to create the ALB.

1. Configure IAM Permissions
Bash

# Download IAM Policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

# Create the Policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
    
Step 2: Update your Environment Variables
Once you have the correct name and region from Step 1, update your session variables. (Replace the values below with what you found in Step 1):

Bash

export MY_CLUSTER_NAME="the-name-from-step-1"
export MY_REGION="us-east-1"

Run this to get the ARN:

Bash

export POLICY_ARN=$(aws iam list-policies --query 'Policies[?PolicyName==`AWSLoadBalancerControllerIAMPolicy`].Arn' --output text)
echo "Policy ARN is: $POLICY_ARN"
🛠️ Fix 2: Set the AWS Region
The eksctl error "AWS Region must be set" happens because your shell environment doesn't know which region your cluster is in.

Run this (Replace us-east-1 if your cluster is elsewhere):

Bash

export AWS_REGION="us-east-1"
# Also set it in your AWS config as a backup
aws configure set region $AWS_REGION

🛠️ Step 1: Enable the OIDC Provider
Run the command that eksctl just suggested. This only needs to be done once per cluster.

Bash

eksctl utils associate-iam-oidc-provider \
  --region=$MY_REGION \
  --cluster=$MY_CLUSTER_NAME \
  --approve
  
🚀 Step 3: Re-run the Service Account Creation
Now that the variables are set and we are using the existing policy, run the eksctl command again:

Bash

eksctl create iamserviceaccount \
  --cluster=spin-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=$POLICY_ARN \
  --region=$AWS_REGION \
  --approve
  
🛠️ Step 1: Install cert-manager
The controller uses certificates for its internal webhooks. Cert-manager is the standard way to handle this.

Bash

kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.13.3/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=available --timeout=300s deployment/cert-manager-webhook -n cert-manager
🛠️ Step 2: Download & Prepare the Controller Manifest
You need to download the official YAML and inject your Cluster Name.

Download the manifest:

Bash

# Replace v2.11.0 with the latest stable if needed
curl -Lo v2_11_0_full.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.11.0/v2_11_0_full.yaml
Edit the file to set your Cluster Name: Open the file: nano v2_11_0_full.yaml Search for --cluster-name (it will be under the Deployment section) and change your-cluster-name to spin-cluster.

Alternatively, use this one-liner to edit it:

Bash

sed -i 's/your-cluster-name/spin-cluster/g' v2_11_0_full.yaml
Remove the ServiceAccount from the YAML: Since you already created the iamserviceaccount using eksctl in the previous step, you must delete the ServiceAccount section from this YAML file so it doesn't overwrite your IAM-linked one.

Find the block starting with apiVersion: v1 and kind: ServiceAccount.

Delete that block (usually the first few lines of the file).

🛠️ Step 3: Apply the Manifests
Now apply the configuration to your cluster.

Bash

# 1. Apply the CRDs and Controller
kubectl apply -f v2_11_0_full.yaml

# 2. Apply the IngressClass and Params (Required for ALB to work)
curl -Lo v2_11_0_ingclass.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.11.0/v2_11_0_ingclass.yaml
kubectl apply -f v2_11_0_ingclass.yaml
🔗 Phase 4: Final Verification
Check that the pods are running correctly:

Bash

kubectl get deployment -n kube-system aws-load-balancer-controller
Why kubectl instead of helm?
Using kubectl gives you a flat YAML file that you can store in your own Git repository (GitOps). However, remember that you are now responsible for manually updating the --cluster-name and ensuring the ServiceAccount doesn't conflict with your eksctl role.

🛠️ Step 1: Get your ALB DNS Name
Go to the EC2 Console -> Load Balancers.

Select your Spinnaker ALB.

Copy the DNS name from the "Details" tab.

🛠️ Step 2: Update Halyard with that DNS Name
Now, use that exact DNS name in your hal commands. This ensures that the UI (Deck) knows how to reach the API (Gate) through the official AWS entry point.

Bash

# Set your new ALB DNS as a variable
export ALB_DNS=$(kubectl get ingress spinnaker-ingress -n spinnaker -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# 1. Update UI URL (No port needed if using Ingress on port 80)
hal config security ui edit --override-base-url http://$ALB_DNS

# 2. Update API URL
# NOTE: If your Ingress uses the "/api" path (as in the YAML), add /api here
hal config security api edit --override-base-url http://$ALB_DNS/api

# 3. Update CORS to allow the UI to talk to the API
hal config security api edit --cors-access-allowed-origin http://$ALB_DNS

🛠️ The Fix: Permission & Path Alignment
Follow these steps to fix the ownership and tell Halyard exactly where the file is.

1. Move the Kubeconfig to a Shared Location
Halyard sometimes struggles with files hidden in a user's home directory. Moving it to /home/spinnaker (the default Halyard user) or ensuring the permissions are open is the best fix.

Bash

# Create a directory Halyard can definitely access
sudo mkdir -p /home/spinnaker/.kube

# Copy your config there
sudo cp /home/ubuntu/.kube/config /home/spinnaker/.kube/config

# Give ownership to the spinnaker user (who runs the halyard daemon)
sudo chown -R spinnaker:spinnaker /home/spinnaker/.kube
sudo chmod 600 /home/spinnaker/.kube/config
2. Update Halyard to use the New Path
Now, tell Spinnaker to look at this new path instead of the one in the ubuntu folder.

Bash

# Update the kubernetes account to use the new file path
hal config provider kubernetes account edit my-k8s-v2-account \
    --kubeconfig-file /home/spinnaker/.kube/config

# Verify the change
hal config provider kubernetes account get my-k8s-v2-account
3. Fix Local Permissions for the Current User
If you want to keep using the file in your ubuntu home, you must ensure the Halyard daemon can read it:

Bash

sudo chmod 755 /home/ubuntu
sudo chmod 755 /home/ubuntu/.kube
sudo chmod 644 /home/ubuntu/.kube/config

🛠️ The Fix: Share AWS Credentials with Halyard
You need to copy your AWS credentials to the spinnaker user's home directory so the daemon can authenticate with EKS.

1. Copy AWS Config to Spinnaker User
Run these commands to give the Halyard daemon the credentials it needs:

Bash

# Create the .aws directory for the spinnaker user
sudo mkdir -p /home/spinnaker/.aws

# Copy your working credentials over
sudo cp -r /home/ubuntu/.aws/* /home/spinnaker/.aws/

# Fix ownership so the spinnaker daemon can read them
sudo chown -R spinnaker:spinnaker /home/spinnaker/.aws
sudo chmod 600 /home/spinnaker/.aws/credentials
2. Restart the Halyard Daemon
The daemon needs to pick up the new environment/files.

Bash

sudo service halyard restart
# Wait 15 seconds for it to fully initialize
sleep 15
3. Verify the Connection
Before running the full deploy, test if Halyard can now "see" the cluster using those credentials:

Bash

hal config provider kubernetes account get my-k8s-v2-account

4. Deploy Spinnaker
Bash

hal config version edit --version 2025.0.1
hal deploy apply

Phase 4: Create the AWS ALB Ingress
This manifest links the external URLs to the internal Spinnaker services.

Create spinnaker-aws-ingress.yaml:

YAML

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: spinnaker-ingress
  namespace: spinnaker
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: spinnaker-lb
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /health
spec:
  ingressClassName: alb
  rules:
  - host: spinnaker.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: spin-deck
            port:
              number: 9000
  - host: spinnaker-api.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: spin-gate
            port:
              number: 8084
Apply it: kubectl apply -f spinnaker-aws-ingress.yaml

Wait about 2 minutes, then run this command to get the assigned DNS Name:

Bash

kubectl get ingress spinnaker-ingress -n spinnaker -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
It will look something like: k8s-spinnaker-lb-xxxxxx.us-east-1.elb.amazonaws.com

🔗 Phase 5: DNS & Access
Get ALB Address: Run kubectl get ingress -n spinnaker. Copy the long address in the ADDRESS column.

Update DNS: Create CNAME records in your DNS provider (e.g., Route53) pointing spinnaker and spinnaker-api to that ALB address.

Check Status: Wait ~3 minutes for the ALB health checks to pass.
