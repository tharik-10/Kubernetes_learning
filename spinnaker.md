1. Prerequisites (AWS Specific)
Instance: t3.xlarge (4 vCPU, 16GB RAM) minimum.

Security Group: Inbound rules for Port 9000 (UI) and 8084 (API) set to 0.0.0.0/0.

OS: Ubuntu 20.04/22.04 LTS.

2. Installation & Core Config
This part follows the standard Halyard flow but adds the "Cloud Fixes" early on.

Bash

# Install Halyard
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh

# Basic Spinnaker Config
hal config provider kubernetes enable
hal config deploy edit --type localdebian
hal config storage edit --type redis

# Select Version
hal config version edit --version $(hal version list | grep -m 1 "" | awk '{print $2}')
3. The "EC2 Public Access" Workflow
This is the most critical step that differs from the local laptop setup. These commands ensure you don't need hal deploy connect.

Bash

# 1. Force all microservices to listen globally (not just localhost)
hal config edit --host 0.0.0.0

# 2. Set the public endpoints so the UI knows how to talk to the API
# Replace [PUBLIC_IP] with your actual EC2 Public IP
hal config security ui edit --override-base-url http://[PUBLIC_IP]:9000
hal config security api edit --override-base-url http://[PUBLIC_IP]:8084

# 3. Deploy
sudo hal deploy apply
4. Post-Deployment "Service Fixes"
Halyard's "Local Debian" mode often leaves the webserver (Apache) locked to localhost. You must verify and fix this manually.

Fix Apache Ports:

Bash

# Ensure Apache listens on all interfaces globally
sudo sed -i 's/Listen 127.0.0.1:9000/Listen 0.0.0.0:9000/g' /etc/apache2/ports.conf
sudo systemctl restart apache2
Fix Spinnaker Site Config:

Bash

# Ensure the VirtualHost is open
sudo sed -i 's/<VirtualHost localhost:9000>/<VirtualHost *:9000>/g' /etc/apache2/sites-available/spinnaker.conf
sudo systemctl restart apache2
sudo systemctl daemon-reload
5. Summary: Why this Workflow works
Feature	Standard "Local" Workflow	Correct "EC2" Workflow
Binding	127.0.0.1 (Private)	0.0.0.0 (Public)
Access	Requires hal deploy connect tunnel	Direct access via Browser
Endpoint	http://localhost:9000	http://[EC2_IP]:9000
Apache	Managed solely by Halyard	Managed by Halyard + Manual Overrides

Export to Sheets

6. What's Next?
Now that your "Control Plane" is up, Spinnaker is essentially a brain without a body. To make it useful, you should:

Add a Kubernetes Account: Use hal config provider kubernetes account add... to give Spinnaker a place to deploy containers.

Enable Artifacts: If you want to use GitHub or S3 to trigger pipelines, run hal config features edit --artifacts true.

Setup CloudProviders: If you are deploying to AWS EC2 directly (not K8s), you'll need to configure the Clouddriver service with AWS IAM credentials.

Would you like the specific commands to connect your first Kubernetes cluster to this Spinnaker instance?

Moving from a Local Debian (single machine) install to a Distributed installation is a major architectural jump. In a distributed setup, Halyard installs Spinnaker’s microservices as individual deployments into a Kubernetes (K8S) cluster.This is the recommended way to run Spinnaker for production because it is self-healing, scalable, and easier to manage long-term.The Distributed Workflow (Step-by-Step)1. PrerequisitesA Kubernetes Cluster: You need an existing K8s cluster (EKS, GKE, or a self-managed one).Kubeconfig: Your EC2 instance must have kubectl installed and configured to talk to that cluster.Cloud Storage: Redis is no longer enough. You should use AWS S3 or Google Cloud Storage for persistent data.2. Point Halyard to KubernetesInstead of localdebian, we tell Halyard to target your cluster.Bash# 1. Enable the Kubernetes provider
hal config provider kubernetes enable

# 2. Add your cluster as a Spinnaker account
# Ensure your current kubectl context is pointing to the right cluster
hal config provider kubernetes account add my-k8s-v2-account \
    --provider-version v2 \
    --context $(kubectl config current-context)

# 3. Set the deployment type to distributed
hal config deploy edit --type distributed --account-name my-k8s-v2-account
3. Configure External Storage (S3 Example)In a distributed setup, if a pod restarts, it loses local data. You must use a persistent store like S3.Bash# 1. Provide your AWS credentials to Halyard
hal config storage s3 edit \
    --access-key-id $YOUR_ACCESS_KEY \
    --secret-access-key \
    --region us-east-1

# 2. Set S3 as the storage source
hal config storage edit --type s3
4. The "EC2 to K8s" Networking SetupSince Spinnaker will now be running inside Kubernetes pods, but you are accessing it from outside (via your EC2 IP or a Load Balancer), you must update the endpoints:Bash# Set the UI and API to point to your Load Balancer or EC2 Public IP
hal config security ui edit --override-base-url http://[PUBLIC_IP_OR_DNS]:9000
hal config security api edit --override-base-url http://[PUBLIC_IP_OR_DNS]:8084
5. Deploy to the ClusterWhen you run the command below, Halyard will not install packages on your Ubuntu machine; instead, it will create a series of Pods and Services in your Kubernetes cluster.Bashsudo hal deploy apply
Key Differences in WorkflowFeatureLocal Debian (What you have now)Distributed (Target)Service LocationSystemd processes on UbuntuPods in KubernetesScalingScale the EC2 Instance (Vertical)Scale individual Pods (Horizontal)StorageLocal Redis (Ephemeral)S3 / GCS / Azure Storage (Persistent)Updatessudo apt-get upgradehal deploy apply (Rolling updates)Important Note on NetworkingIn a Distributed setup, Port 9000 and 8084 won't automatically be open. You will need to change the Kubernetes service type for spin-deck (UI) and spin-gate (API) to LoadBalancer or NodePort so you can reach them from your browser.
