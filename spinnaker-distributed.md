🏗️ Phase 1: Cluster & CLI SetupThis phase prepares your management machine and builds the EKS cluster.1. Install Required ToolsBash# Install AWS CLI, Kubectl, and Eksctl
sudo apt update && sudo apt install -y docker.io unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/eksctl
2. AWS AuthenticationOption A: IAM Role (Recommended)Attach an IAM Role with AdministratorAccess to your EC2 instance (Actions > Security > Modify IAM role).Option B: Access KeysBashaws configure
# Enter Access Key, Secret Key, and Region (e.g., us-east-1)
3. Create the EKS ClusterBasheksctl create cluster \
  --name spin-cluster \
  --region us-east-1 \
  --nodegroup-name standard-nodes \
  --node-type t3.xlarge \
  --nodes 3 \
  --with-oidc \
  --managed
🛠️ Phase 2: Halyard & Spinnaker DeploymentSpinnaker runs as a set of microservices managed by the Halyard daemon.1. Fix Permissions for HalyardHalyard runs as the spinnaker user and needs access to your kubeconfig and AWS credentials.Bash# Create directories
sudo mkdir -p /home/spinnaker/.kube /home/spinnaker/.aws

# Copy configs
sudo cp ~/.kube/config /home/spinnaker/.kube/config
sudo cp -r ~/.aws/* /home/spinnaker/.aws/

# Fix ownership
sudo chown -R spinnaker:spinnaker /home/spinnaker/.kube /home/spinnaker/.aws
sudo chmod 600 /home/spinnaker/.kube/config /home/spinnaker/.aws/credentials

# Restart Halyard to pick up changes
sudo service halyard restart && sleep 15
2. Configure Kubernetes ProviderBashhal config provider kubernetes enable
hal config provider kubernetes account add my-k8s-account \
    --context $(kubectl config current-context) \
    --kubeconfig-file /home/spinnaker/.kube/config
hal config deploy edit --type distributed --account-name my-k8s-account
3. Deploy SpinnakerBashhal config version edit --version 2025.0.1
hal deploy apply
🌐 Phase 3: Accessing SpinnakerChoose ONE of the following methods to access the UI.Option A: NodePort (Easiest / Free)This uses your Worker Node's Public IP and high-range ports.1. Change Service TypesBashkubectl patch svc spin-deck -n spinnaker -p '{"spec": {"type": "NodePort"}}'
kubectl patch svc spin-gate -n spinnaker -p '{"spec": {"type": "NodePort"}}'
2. Get your Public IP and PortsBash# Get the ports (e.g., 9000:32410 and 8084:30541)
kubectl get svc -n spinnaker

# Get Public IP of a node
aws ec2 describe-instances --filters "Name=tag:kubernetes.io/cluster/spin-cluster,Values=owned" --query "Reservations[*].Instances[*].PublicIpAddress" --output text
3. CRITICAL: Update AWS Security GroupGo to EC2 Console > Instances > Select an EKS Node.Under Security, click the Security Group.Edit Inbound Rules and add:TCP 32410 (Deck) from 0.0.0.0/0TCP 30541 (Gate) from 0.0.0.0/04. Update Halyard URLsBashexport NODE_IP="[YOUR_PUBLIC_IP]"
hal config security ui edit --override-base-url http://$NODE_IP:32410
hal config security api edit --override-base-url http://$NODE_IP:30541
hal config security api edit --cors-access-pattern http://$NODE_IP:32410
hal deploy apply
Option B: ALB Ingress Controller (Professional)This creates a single AWS Application Load Balancer.1. Install LB Controller & Cert-ManagerBash# Install Cert-Manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.13.3/cert-manager.yaml

# Create IAM Service Account for Controller
eksctl create iamserviceaccount \
  --cluster=spin-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonEKSLoadBalancerControllerIAMPolicy \
  --approve
2. Apply Ingress ManifestCreate spinnaker-ingress.yaml:YAMLapiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: spinnaker-ingress
  namespace: spinnaker
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: spin-deck
            port:
              number: 9000
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: spin-gate
            port:
              number: 8084
kubectl apply -f spinnaker-ingress.yaml3. Update Halyard for ALBBashexport ALB_URL=$(kubectl get ingress spinnaker-ingress -n spinnaker -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

hal config security ui edit --override-base-url http://$ALB_URL
hal config security api edit --override-base-url http://$ALB_URL/api/v1
hal deploy apply
🔍 Phase 4: Troubleshooting Common IssuesIssueCauseFix"Error fetching applications"CORS or Port mismatchEnsure hal config security api edit --cors-access-pattern matches the UI URL exactly.404 on API callsPath mismatch in IngressEnsure your Ingress path (/api/v1) matches the Halyard override-base-url.UI spins foreverSecurity Group blockedVerify the Gate port (e.g., 30541) is open in AWS EC2 Security Groups."Halyard cannot connect"Permission issueRun the chown -R spinnaker:spinnaker commands in Phase 2 again.
