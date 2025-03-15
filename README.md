# Kubernetes Voting App Deployment on Azure Kubernetes Service (AKS)

This guide provides step-by-step instructions to deploy a cloud-native voting application on Azure Kubernetes Service (AKS). The application consists of a frontend, a backend API, and a MongoDB database. The frontend allows users to vote for their favorite tools, the API processes the votes, and MongoDB stores the voting data.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Step 1: Create an Azure Resource Group](#step-1-create-an-azure-resource-group)
3. [Step 2: Create an AKS Cluster](#step-2-create-an-aks-cluster)
4. [Step 3: Configure kubectl to Connect to AKS](#step-3-configure-kubectl-to-connect-to-aks)
5. [Step 4: Verify the Cluster](#step-4-verify-the-cluster)
6. [Step 5: Clone the GitHub Repository](#step-5-clone-the-github-repository)
7. [Step 6: Create a Kubernetes Namespace](#step-6-create-a-kubernetes-namespace)
8. [Step 7: Deploy MongoDB](#step-7-deploy-mongodb)
   - [Create MongoDB Deployment](#create-mongodb-deployment)
   - [Verify MongoDB DNS Resolution](#verify-mongodb-dns-resolution)
   - [Initialize MongoDB Replica Set](#initialize-mongodb-replica-set)
9. [Step 8: Load Data into MongoDB](#step-8-load-data-into-mongodb)
10. [Step 9: Deploy the API](#step-9-deploy-the-api)
11. [Step 10: Get API Endpoint and Test](#step-10-get-api-endpoint-and-test)
12. [Step 11: Deploy the Frontend](#step-11-deploy-the-frontend)
13. [Step 12: Get Frontend URL](#step-12-get-frontend-url)
14. [Step 13: Test the Application](#step-13-test-the-application)
15. [Step 14: Query MongoDB for Updated Votes](#step-14-query-mongodb-for-updated-votes)
16. [Cleanup Resources](#cleanup-resources)

---

## Prerequisites

Before proceeding, ensure you have the following tools installed and configured:

1. **Azure CLI**: Install and configure Azure CLI.
   - Install Azure CLI using PowerShell:
     ```powershell
     Start-Process "https://aka.ms/installazurecliwindows"
     ```
   - Follow the installation wizard to complete the setup.
   - Verify installation:
     ```powershell
     az --version
     ```
   - Log in to Azure:
     ```powershell
     az login --allow-no-subscriptions
     ```

2. **kubectl**: Install Kubernetes CLI.
   - Install kubectl:
     ```sh
     curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
     chmod +x kubectl
     sudo mv kubectl /usr/local/bin/
     ```
   - Verify installation:
     ```sh
     kubectl version --client
     ```

3. **Helm**: Install Helm for package management.
   - Install Helm:
     ```sh
     curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
     ```
   - Verify installation:
     ```sh
     helm version
     ```

4. **Azure Subscription**: Ensure you have an active Azure subscription.

5. **Git**: Install Git for cloning the repository.
   - Install Git:
     ```sh
     sudo apt-get install git
     ```
   - Verify installation:
     ```sh
     git --version
     ```

---

## Step 1: Create an Azure Resource Group

Create a resource group to organize your AKS resources.

```sh
az group create --name AKSResourceGroup --location eastus
```

---

## Step 2: Create an AKS Cluster

Create an AKS cluster with a single node.

```sh
az aks create --resource-group AKSResourceGroup --name MyAKSCluster --node-count 1 --node-vm-size Standard_B2s --enable-addons monitoring --generate-ssh-keys --tier free
```

---

## Step 3: Configure kubectl to Connect to AKS

Configure `kubectl` to connect to your AKS cluster.

```sh
az aks get-credentials --resource-group AKSResourceGroup --name MyAKSCluster
```

---

## Step 4: Verify the Cluster

Verify that the cluster is running and accessible.

```sh
kubectl get nodes
```

---

## Step 5: Clone the GitHub Repository

Clone the repository containing the Kubernetes manifests for the voting app.

```sh
git clone https://github.com/chidhvilaskacham/k8s-voteapp.git
cd k8s-voteapp
```

---

## Step 6: Create a Kubernetes Namespace

Create a namespace for the voting app.

```sh
kubectl create ns k8s-voteapp
kubectl config set-context --current --namespace k8s-voteapp
```

---

## Step 7: Deploy MongoDB

### Create MongoDB Deployment

Deploy MongoDB using the provided manifest.

```sh
kubectl apply -f mongo-deployment.yaml
```

### Verify MongoDB DNS Resolution

Verify that the MongoDB Pods can resolve each other's DNS names.

```sh
kubectl run --rm utils -it --image praqma/network-multitool -- bash
for i in {0..2}; do nslookup mongo-$i.mongo; done
exit
```

### Initialize MongoDB Replica Set

Initialize the MongoDB replica set.

```powershell
# Wait for the Pods to be ready
Write-Host "Waiting for MongoDB Pods to be ready..."
Start-Sleep -Seconds 30

# Initialize the replica set
Write-Host "Initializing MongoDB Replica Set..."
kubectl exec -it mongo-0 -n k8s-voteapp -- mongo --username admin --password password --authenticationDatabase admin --eval "rs.initiate()"
Start-Sleep -Seconds 2

# Add the first member (mongo-1)
kubectl exec -it mongo-0 -n k8s-voteapp -- mongo --username admin --password password --authenticationDatabase admin --eval 'rs.add("mongo-1.mongo-service:27017")'
Start-Sleep -Seconds 2

# Add the second member (mongo-2)
kubectl exec -it mongo-0 -n k8s-voteapp -- mongo --username admin --password password --authenticationDatabase admin --eval 'rs.add("mongo-2.mongo-service:27017")'
Start-Sleep -Seconds 2

# Reconfigure the replica set
kubectl exec -it mongo-0 -n k8s-voteapp -- mongo --username admin --password password --authenticationDatabase admin --eval 'cfg = rs.conf(); cfg.members[0].host = "mongo-0.mongo-service:27017"; rs.reconfig(cfg, {force: true})'
Start-Sleep -Seconds 5

# Verify the replica set status
Write-Host "Verifying Replica Set Status..."
kubectl exec -it mongo-0 -n k8s-voteapp -- mongo --username admin --password password --authenticationDatabase admin --eval "rs.status()"
```

---

## Step 8: Load Data into MongoDB

Load initial data into MongoDB.

```sh
kubectl exec -it mongo-0 -- mongo <<EOF
use langdb;
tools = {
    "Ansible": {"uses": "Configuration management, automation", "votes": 0},
    "Visual_studio": {"uses": "IDE for software development", "votes": 0},
    "Docker": {"uses": "Containerization platform", "votes": 0},
    "Prometheus": {"uses": "Monitoring and alerting toolkit", "votes": 0},
    "Git": {"uses": "Version control system", "votes": 0},
    "Jenkins": {"uses": "Continuous integration and delivery", "votes": 0}
};
for (var tool in tools) {
    db.languages.insert({"name": tool, "details": tools[tool]});
}
db.languages.find().pretty();
EOF
```

---

## Step 9: Deploy the API

Deploy the backend API.

```sh
kubectl apply -f api-deployment.yaml
```

---

## Step 10: Get API Endpoint and Test

Get the API endpoint and test it.

```sh
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
curl $API_ELB_PUBLIC_FQDN/ok
curl -s $API_ELB_PUBLIC_FQDN/languages | jq .
```

---

## Step 11: Deploy the Frontend

Deploy the frontend.

```sh
kubectl apply -f frontend-deployment.yaml
```

---

## Step 12: Get Frontend URL

Get the frontend URL.

```sh
FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
echo http://$FRONTEND_ELB_PUBLIC_FQDN
```

---

## Step 13: Test the Application

Open the frontend URL in a browser and interact with the voting application.

---

## Step 14: Query MongoDB for Updated Votes

Query MongoDB to see updated votes.

```sh
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```

---

## Cleanup Resources

Delete the AKS cluster and resource group to avoid unnecessary charges.

```sh
az aks delete --name MyAKSCluster --resource-group AKSResourceGroup --yes --no-wait
az group delete --name AKSResourceGroup --yes --no-wait
```

---

## Conclusion

You have successfully deployed a cloud-native voting application on Azure Kubernetes Service (AKS). This setup includes a frontend, a backend API, and a MongoDB database, all running in a Kubernetes cluster. You can now scale the application, monitor its performance, and manage it using Kubernetes tools.

For further improvements, consider:
- Adding monitoring and logging using Azure Monitor or Prometheus.
- Implementing CI/CD pipelines for automated deployments.
- Securing the application with network policies and HTTPS.

Happy voting! 🚀
