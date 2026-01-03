🚀 Complete Guide: Deploying Nginx via Spinnaker on AWS EKSPhase 1: Fixing the "Forbidden" Permissions (RBAC)Spinnaker needs "God Mode" (Cluster-Admin) to manage your EKS cluster. The errors we saw occurred because the Ubuntu node's IAM role wasn't mapped correctly to Kubernetes.Run these on your Ubuntu terminal:Correct the IAM Mapping:Fix the typo in the aws-auth ConfigMap to grant your node admin rights.Basheksctl create iamidentitymapping \
    --cluster demo-eks \
    --region us-east-1 \
    --arn arn:aws:iam::832871077896:role/eksctl-demo-eks-nodegroup-spinnake-NodeInstanceRole-elKnc7pIcbE5 \
    --group system:masters \
    --username admin \
    --force
Clear Spinnaker's Cache:Forcing a restart ensures Spinnaker picks up the new permissions immediately.Bashkubectl rollout restart deployment spin-clouddriver -n spinnaker
Phase 2: The Full "Working" ManifestThis YAML includes the Moniker annotations (to make the green boxes appear in the Clusters UI) and the AWS Load Balancer annotations (to ensure the site is accessible from the internet).In Spinnaker, go to your "Deploy (Manifest)" stage and paste this:YAMLapiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-simple
  namespace: default
  annotations:
    # Essential for Spinnaker UI visibility
    moniker.spinnaker.io/application: nginx-simple
    moniker.spinnaker.io/cluster: nginx-simple
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-lb
  namespace: default
  annotations:
    # Forces AWS to create an Internet-facing Load Balancer
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
Phase 3: Final Infrastructure Checks (The "Access" Step)Even with the right YAML, AWS firewalls (Security Groups) can still block your browser.Run the Pipeline: Trigger a manual execution. It should now turn Green.Get your URL:Bashkubectl get svc nginx-service-lb -n default
Open AWS Inbound Rules:Go to EC2 > Load Balancers in the AWS Console.Find the Security Group for your new LB.Crucial Step: Add an Inbound Rule for HTTP (Port 80) from source 0.0.0.0/0.Phase 4: Error Troubleshooting Cheat SheetError MessageMeaningFixForbidden... system:node...Spinnaker node has no admin rights.Run the eksctl create iamidentitymapping command above.Taking too long to respondFirewall is blocking your browser.Add Port 80 to the Load Balancer's Security Group in AWS.Internal FacingLB is only for private VPC use.Add the internet-facing annotation to your Service manifest.Clusters tab is emptyMissing Moniker annotations.Add moniker.spinnaker.io/application to your Deployment metadata.Final VerificationGreen Pods: Check the Clusters tab. You should see a server group named nginx-simple-v001.Active LB: Check the Load Balancers tab. It should show your DNS name.Live Site: Paste the DNS name into your browser and see "Welcome to nginx!".
