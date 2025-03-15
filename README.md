# Deploying a Kubernetes Voting App on Azure Kubernetes Service (AKS)

This guide provides steps to deploy a cloud-native voting application on Azure Kubernetes Service (AKS). The application consists of a frontend, a backend API, and a MongoDB database.

## Prerequisites

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

4. **Azure Subscription**: Ensure you have an active Azure subscription.

5. **Git**: Install Git for cloning the repository.

## Step 1: Create an Azure Resource Group

```sh
az group create --name AKSResourceGroup --location eastus
```

## Step 2: Create an AKS Cluster

```sh
az aks create --resource-group AKSResourceGroup --name MyAKSCluster --node-count 1 --node-vm-size Standard_B2s --enable-addons monitoring --generate-ssh-keys --tier free
```

## Step 3: Configure kubectl to Connect to AKS

```sh
az aks get-credentials --resource-group AKSResourceGroup --name MyAKSCluster
```

## Step 4: Verify the Cluster

```sh
kubectl get nodes
```

## Step 5: Clone the GitHub Repository

```sh
git clone https://github.com/chidhvilaskacham/k8s-voteapp.git
cd K8s-voting-app
```

## Step 6: Create a Kubernetes Namespace

```sh
kubectl create ns k8s-voteapp
kubectl config set-context --current --namespace k8s-voteapp
```

## Step 7: Deploy MongoDB

### Create MongoDB Deployment

```sh
kubectl apply -f mongo-deployment.yaml
```

### Verify MongoDB DNS Resolution

```sh
kubectl run --rm utils -it --image praqma/network-multitool -- bash
for i in {0..2}; do nslookup mongo-$i.mongo; done
exit
```

### Initialize MongoDB Replica Set


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

```sh
kubectl exec -it mongo-0 -- mongo --eval "rs.status()" | grep "PRIMARY\|SECONDARY"
```

## Step 8: Load Data into MongoDB

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

## Step 9: Deploy the API

```sh
kubectl apply -f api-deployment.yaml
```

## Step 10: Get API Endpoint and Test

```sh
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
curl $API_ELB_PUBLIC_FQDN/ok
curl -s $API_ELB_PUBLIC_FQDN/languages | jq .
```

## Step 11: Deploy the Frontend

```sh
kubectl apply -f frontend-deployment.yaml
```

## Step 12: Get Frontend URL

```sh
FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
echo http://$FRONTEND_ELB_PUBLIC_FQDN
```

## Step 13: Test the Application

Open the frontend URL in a browser and interact with the voting application.

## Step 14: Query MongoDB for Updated Votes

```sh
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```

## Cleanup Resources

```sh
az aks delete --name MyAKSCluster --resource-group AKSResourceGroup --yes --no-wait
az group delete --name AKSResourceGroup --yes --no-wait
```


