Step 1: Run a Pod with the Nginx image

Create a simple Pod running the nginx container:

kubectl run nginx --image=nginx


✅ This command:

Creates a Pod named nginx

Uses the Docker Hub nginx image (latest version)

Starts a single container inside it

🧩 Step 2: Verify the Pod

Check if it’s running:

kubectl get pods


If it’s stuck in ContainerCreating or ImagePullBackOff, check details:

kubectl describe pod nginx

🧩 Step 3: Expose the Pod (for external access)

By default, Pods are only reachable inside the cluster.
To expose it as a Service, use:

kubectl expose pod nginx --port=80 --type=NodePort


✅ This command:

Creates a Service named nginx

Opens port 80 on the Pod

Assigns a NodePort (a high external port, e.g., 30080)

Check the Service:

kubectl get svc


You’ll see something like:

NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.96.213.57    <none>        80:30080/TCP   1m


Now, you can access it using:

curl http://<Node-IP>:<NodePort>


Example:

curl http://192.168.49.2:30080

🧩 Step 4: Run as a Deployment (recommended)

Pods alone aren’t self-healing or scalable.
So, instead, run an Nginx Deployment:

kubectl create deployment nginx-deploy --image=nginx


Check:

kubectl get deployments


View Pods created by Deployment:

kubectl get pods -l app=nginx-deploy

🧩 Step 5: Expose the Deployment

Make the Deployment accessible externally:

kubectl expose deployment nginx-deploy --port=80 --type=NodePort


Check the Service:

kubectl get svc

🧩 Step 6: Scale it up (optional)

To increase the number of replicas:

kubectl scale deployment nginx-deploy --replicas=3


Verify:

kubectl get pods

🔍 Summary
Action	Command
Run Nginx Pod	kubectl run nginx --image=nginx
Expose Pod	kubectl expose pod nginx --port=80 --type=NodePort
Create Deployment	kubectl create deployment nginx-deploy --image=nginx
Expose Deployment	kubectl expose deployment nginx-deploy --port=80 --type=NodePort
Scale Deployment	kubectl scale deployment nginx-deploy --replicas=3
