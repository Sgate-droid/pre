# pre

### Step-by-Step Guide to Install and Configure a 3-Node HA Cluster with PostgreSQL Database and Apache on Kubernetes

This guide will walk you through setting up a 3-node High Availability (HA) cluster using Kubernetes. The setup will include:
1. A 3-node Kubernetes cluster.
2. PostgreSQL database deployed as containers.
3. Apache web server deployed on Kubernetes nodes.

---

### **Prerequisites**
1. **Three Linux servers** (Ubuntu 20.04/22.04 recommended) with:
   - Minimum 2 CPU cores, 4GB RAM, and 20GB disk space per node.
   - Static IP addresses for all nodes.
   - SSH access to all nodes.
2. **kubectl** installed on your local machine.
3. **Docker** installed on all nodes.
4. **Helm** installed on your local machine (for deploying PostgreSQL).

---

### **Step 1: Set Up a 3-Node Kubernetes Cluster**

#### 1.1 Install Kubernetes on All Nodes
Run the following commands on all three nodes:

```bash
# Update the system
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker

# Add Kubernetes repository
sudo apt install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update

# Install kubeadm, kubectl, and kubelet
sudo apt install -y kubeadm kubectl kubelet
sudo apt-mark hold kubeadm kubectl kubelet
```

#### 1.2 Initialize the Kubernetes Cluster on the Master Node
On the **master node**, run:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

After initialization, you'll see a `kubeadm join` command. Save this command for joining the worker nodes.

#### 1.3 Set Up kubectl on the Master Node
Run the following commands on the master node:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 1.4 Join Worker Nodes to the Cluster
Run the `kubeadm join` command (from Step 1.2) on both worker nodes.

```bash
kubeadm join <master-ip>:<port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

#### 1.5 Install a Network Plugin (Calico)
On the master node, install Calico for networking:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

#### 1.6 Verify the Cluster
On the master node, check the status of the nodes:

```bash
kubectl get nodes
```

All nodes should be in the `Ready` state.

---

### **Step 2: Deploy PostgreSQL as Containers Using Helm**

#### 2.1 Install Helm
On your local machine, install Helm:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

#### 2.2 Add the Bitnami Helm Repository
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

#### 2.3 Deploy PostgreSQL
Deploy a PostgreSQL HA cluster using Helm:

```bash
helm install postgres-ha bitnami/postgresql-ha \
  --set postgresql.replicaCount=3 \
  --set postgresql.password=yourpassword \
  --set postgresql.database=myappdb
```

Replace `yourpassword` with a secure password.

#### 2.4 Verify PostgreSQL Deployment
Check the status of the PostgreSQL pods:

```bash
kubectl get pods -l app=postgresql-ha
```

---

### **Step 3: Deploy Apache Web Server on Kubernetes**

#### 3.1 Create a Deployment for Apache
Create a file named `apache-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache
spec:
  replicas: 3
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
      - name: apache
        image: httpd:latest
        ports:
        - containerPort: 80
```

Apply the deployment:

```bash
kubectl apply -f apache-deployment.yaml
```

#### 3.2 Expose Apache as a Service
Create a file named `apache-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: apache
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  selector:
    app: apache
```

Apply the service:

```bash
kubectl apply -f apache-service.yaml
```

#### 3.3 Verify Apache Deployment
Check the status of the Apache pods:

```bash
kubectl get pods -l app=apache
```

Access the Apache web server using any node's IP address and port `30080` (e.g., `http://<node-ip>:30080`).

---

### **Step 4: Test High Availability**

#### 4.1 Simulate Node Failure
- Stop one of the worker nodes and verify that the PostgreSQL and Apache services continue to function.
- Use `kubectl get pods -o wide` to see how Kubernetes reschedules pods to healthy nodes.

#### 4.2 Test Database Failover
- Connect to the PostgreSQL database and verify read/write operations.
- Use the following command to connect:

```bash
kubectl run postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:latest --env="PGPASSWORD=yourpassword" --command -- psql --host postgres-ha-postgresql-ha-pgpool -U postgres -d myappdb
```

---

### **Step 5: Monitor the Cluster**

#### 5.1 Install Kubernetes Dashboard (Optional)
Deploy the Kubernetes dashboard for monitoring:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

Access the dashboard using:

```bash
kubectl proxy
```

Then open `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`.

#### 5.2 Use Prometheus and Grafana (Optional)
For advanced monitoring, deploy Prometheus and Grafana using Helm.

---

### **Conclusion**
You now have a 3-node HA Kubernetes cluster with PostgreSQL and Apache deployed. This setup ensures high availability and scalability for your applications.
