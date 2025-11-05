# 🧭 Kubernetes Commands (Basic → Intermediate)

This file contains a complete list of essential `kubectl` commands from **basic to intermediate** with short, one-line explanations.

---

## 🌱 1️⃣ Cluster Information
| Command | Description |
|----------|-------------|
| `kubectl version` | Shows client and server Kubernetes versions |
| `kubectl cluster-info` | Displays cluster control plane and services info |
| `kubectl get componentstatuses` | Lists status of control plane components |
| `kubectl config view` | Shows kubeconfig details |
| `kubectl config current-context` | Displays current Kubernetes context |
| `kubectl config use-context <context>` | Switches between clusters/contexts |

---

## 🧩 2️⃣ Working with Nodes
| Command | Description |
|----------|-------------|
| `kubectl get nodes` | Lists all worker and master nodes |
| `kubectl describe node <node-name>` | Shows detailed info about a specific node |
| `kubectl cordon <node-name>` | Marks node as unschedulable (no new pods) |
| `kubectl drain <node-name>` | Safely evicts pods before maintenance |
| `kubectl uncordon <node-name>` | Makes node schedulable again |

---

## 🧱 3️⃣ Working with Pods
| Command | Description |
|----------|-------------|
| `kubectl run nginx --image=nginx` | Creates a new Pod using Nginx image |
| `kubectl get pods` | Lists all Pods in the current namespace |
| `kubectl describe pod <pod-name>` | Shows detailed Pod info and events |
| `kubectl logs <pod-name>` | Displays logs from a Pod |
| `kubectl exec -it <pod-name> -- /bin/bash` | Enters the Pod’s shell |
| `kubectl delete pod <pod-name>` | Deletes a Pod |
| `kubectl get pods -o wide` | Shows Pod details including IPs and Nodes |

---

## 📦 4️⃣ Working with Deployments
| Command | Description |
|----------|-------------|
| `kubectl create deployment myapp --image=nginx` | Creates a Deployment |
| `kubectl get deployments` | Lists all Deployments |
| `kubectl describe deployment <name>` | Shows detailed info about a Deployment |
| `kubectl scale deployment <name> --replicas=5` | Scales a Deployment |
| `kubectl edit deployment <name>` | Edits Deployment definition live |
| `kubectl delete deployment <name>` | Deletes a Deployment |

---

## 🔁 5️⃣ Rollouts (Updates & Rollbacks)
| Command | Description |
|----------|-------------|
| `kubectl rollout status deployment/<name>` | Shows rollout progress |
| `kubectl rollout history deployment/<name>` | Lists all previous revisions |
| `kubectl rollout undo deployment/<name>` | Rolls back to the previous version |
| `kubectl rollout undo deployment/<name> --to-revision=2` | Rolls back to a specific revision |
| `kubectl rollout pause deployment/<name>` | Pauses the rollout |
| `kubectl rollout resume deployment/<name>` | Resumes a paused rollout |

---

## 🧩 6️⃣ Services & Networking
| Command | Description |
|----------|-------------|
| `kubectl expose pod nginx --port=80 --type=NodePort` | Exposes a Pod as a Service |
| `kubectl get svc` | Lists all Services |
| `kubectl describe svc <service-name>` | Shows details of a Service |
| `kubectl delete svc <service-name>` | Deletes a Service |
| `kubectl port-forward pod/<pod-name> 8080:80` | Forwards local port to Pod port |
| `kubectl get endpoints` | Lists endpoints linked to Services |

---

## 🧭 7️⃣ ConfigMaps & Secrets
| Command | Description |
|----------|-------------|
| `kubectl create configmap my-config --from-literal=key=value` | Creates a ConfigMap |
| `kubectl get configmaps` | Lists all ConfigMaps |
| `kubectl describe configmap my-config` | Shows ConfigMap details |
| `kubectl create secret generic my-secret --from-literal=password=123` | Creates a Secret |
| `kubectl get secrets` | Lists all Secrets |
| `kubectl describe secret my-secret` | Shows Secret details (encoded) |

---

## ⚙️ 8️⃣ Namespaces
| Command | Description |
|----------|-------------|
| `kubectl get ns` | Lists all namespaces |
| `kubectl create ns dev` | Creates a new namespace |
| `kubectl delete ns dev` | Deletes a namespace |
| `kubectl get pods -n dev` | Lists Pods in a specific namespace |
| `kubectl config set-context --current --namespace=dev` | Sets default namespace |

---

## 🧠 9️⃣ Resource Management
| Command | Description |
|----------|-------------|
| `kubectl get all` | Lists all resources in the namespace |
| `kubectl describe <resource> <name>` | Describes any resource (e.g., pod, svc) |
| `kubectl delete -f file.yaml` | Deletes resources defined in a YAML file |
| `kubectl apply -f file.yaml` | Applies configuration changes |
| `kubectl create -f file.yaml` | Creates resources from YAML |
| `kubectl explain <resource>` | Shows documentation for any resource |
| `kubectl get <resource> -o yaml` | Displays resource configuration in YAML |

---

## 🧰 🔟 Debugging & Troubleshooting
| Command | Description |
|----------|-------------|
| `kubectl logs <pod>` | Shows container logs |
| `kubectl logs <pod> -c <container>` | Logs for multi-container Pod |
| `kubectl exec -it <pod> -- /bin/sh` | Opens Pod shell for debugging |
| `kubectl describe <pod>` | Shows detailed Pod info and events |
| `kubectl get events` | Lists cluster events |
| `kubectl top pods` | Shows CPU and memory usage of Pods |
| `kubectl top nodes` | Shows node-level resource usage |

---

## 🧩 11️⃣ Advanced (Intermediate Level)
| Command | Description |
|----------|-------------|
| `kubectl rollout restart deployment/<name>` | Restarts Pods in a Deployment |
| `kubectl get rs` | Lists ReplicaSets |
| `kubectl get daemonsets` | Lists DaemonSets |
| `kubectl get statefulsets` | Lists StatefulSets |
| `kubectl get ingress` | Lists Ingress resources |
| `kubectl describe ingress <name>` | Shows Ingress rules and backend |
| `kubectl get pv,pvc` | Lists PersistentVolumes and Claims |
| `kubectl describe pvc <name>` | Shows PersistentVolumeClaim details |
| `kubectl get serviceaccounts` | Lists Service Accounts |
| `kubectl get roles,rolebindings` | Lists Roles and RoleBindings |

---

## 🧹 12️⃣ Cleanup Commands
| Command | Description |
|----------|-------------|
| `kubectl delete all --all` | Deletes all resources in the namespace |
| `kubectl delete pod --all -n dev` | Deletes all Pods in namespace `dev` |
| `kubectl delete -f <file.yaml>` | Deletes resource from YAML file |

---

## 🧾 Bonus Tip
💡 You can always use:
```bash
kubectl explain <resource> --recursive
```
to see **all fields** and their meanings — super useful when writing YAML files!
