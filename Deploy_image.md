# 🧩 Deploy Nginx on Kubernetes

This guide shows how to **deploy Nginx** on a running Kubernetes cluster using both **Pod** and **Deployment** approaches, and expose it for external access via a **NodePort Service**.

---

## 🚀 Step 1: Run a Pod with the Nginx Image

Create a simple Pod running the Nginx container:

```bash
kubectl run nginx --image=nginx
```

✅ **This command:**
- Creates a Pod named **nginx**
- Uses the Docker Hub **nginx** image (latest version)
- Starts a single container inside it

---

## 🧩 Step 2: Verify the Pod

Check if it’s running:

```bash
kubectl get pods
```

If it’s stuck in `ContainerCreating` or `ImagePullBackOff`, check details:

```bash
kubectl describe pod nginx
```

---

## 🌐 Step 3: Expose the Pod (for External Access)

By default, Pods are only reachable **inside** the cluster.  
To expose it as a Service, use:

```bash
kubectl expose pod nginx --port=80 --type=NodePort
```

✅ **This command:**
- Creates a Service named **nginx**
- Opens **port 80** on the Pod
- Assigns a **NodePort** (a high external port, e.g., `30080`)

Check the Service:

```bash
kubectl get svc
```

Example output:

```
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.96.213.57    <none>        80:30080/TCP   1m
```

Now, access it using:

```bash
curl http://<Node-IP>:<NodePort>
```

Example:

```bash
curl http://192.168.49.2:30080
```

---

## 🧱 Step 4: Run as a Deployment (Recommended)

Pods alone aren’t self-healing or scalable.  
Instead, run an **Nginx Deployment**:

```bash
kubectl create deployment nginx-deploy --image=nginx
```

Check the Deployment:

```bash
kubectl get deployments
```

View Pods created by the Deployment:

```bash
kubectl get pods -l app=nginx-deploy
```

---

## 🌍 Step 5: Expose the Deployment

Make the Deployment accessible externally:

```bash
kubectl expose deployment nginx-deploy --port=80 --type=NodePort
```

Check the Service:

```bash
kubectl get svc
```

---

## ⚙️ Step 6: Scale it Up (Optional)

Increase the number of replicas to handle more load:

```bash
kubectl scale deployment nginx-deploy --replicas=3
```

Verify:

```bash
kubectl get pods
```

---

## 🧾 Summary

| Action | Command |
|--------|----------|
| **Run Nginx Pod** | `kubectl run nginx --image=nginx` |
| **Expose Pod** | `kubectl expose pod nginx --port=80 --type=NodePort` |
| **Create Deployment** | `kubectl create deployment nginx-deploy --image=nginx` |
| **Expose Deployment** | `kubectl expose deployment nginx-deploy --port=80 --type=NodePort` |
| **Scale Deployment** | `kubectl scale deployment nginx-deploy --replicas=3` |

---

✅ **You have successfully:**
- Deployed Nginx as a Pod  
- Exposed it externally using a NodePort Service  
- Created a scalable Nginx Deployment  
- Verified and scaled your application 🚀

