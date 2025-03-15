# Deploying a Kubernetes Voting App on Azure Kubernetes Service (AKS)

This guide provides steps to deploy a cloud-native voting application on Azure Kubernetes Service (AKS). The application consists of a frontend, a backend API, and a MongoDB database.

## Prerequisites

1. **Azure CLI**: Install and configure Azure CLI.
   - Install Azure CLI:
     ```sh
     curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
     ```
   - Verify installation:
     ```sh
     az --version
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
az aks create --resource-group AKSResourceGroup --name MyAKSCluster --node-count 2 --node-vm-size Standard_D2s_v3 --enable-addons monitoring --generate-ssh-keys
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
git clone https://github.com/N4si/K8s-voting-app.git
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
kubectl apply -f manifests/mongo-deployment.yaml
```

### Verify MongoDB DNS Resolution

```sh
kubectl run --rm utils -it --image praqma/network-multitool -- bash
for i in {0..2}; do nslookup mongo-$i.mongo; done
exit
```

### Initialize MongoDB Replica Set

```sh
kubectl exec -it mongo-0 -- mongo <<EOF
rs.initiate();
sleep(2000);
rs.add("mongo-1.mongo:27017");
sleep(2000);
rs.add("mongo-2.mongo:27017");
sleep(2000);
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);
EOF
```

### Verify MongoDB Replica Set

```sh
kubectl exec -it mongo-0 -- mongo --eval "rs.status()" | grep "PRIMARY\|SECONDARY"
```

## Step 8: Load Data into MongoDB

```sh
kubectl exec -it mongo-0 -- mongo <<EOF
use langdb;
db.languages.insert({"name" : "python", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 3, "script" : false, "homepage" : "https://www.python.org/", "download" : "https://www.python.org/downloads/", "votes" : 0}});
db.languages.insert({"name" : "javascript", "codedetail" : { "usecase" : "web, client-side", "rank" : 7, "script" : false, "homepage" : "https://en.wikipedia.org/wiki/JavaScript", "download" : "n/a", "votes" : 0}});
db.languages.find().pretty();
EOF
```

## Step 9: Deploy the API

```sh
kubectl apply -f manifests/api-deployment.yaml
```

## Step 10: Get API Endpoint and Test

```sh
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
curl $API_ELB_PUBLIC_FQDN/ok
curl -s $API_ELB_PUBLIC_FQDN/languages | jq .
```

## Step 11: Deploy the Frontend

```sh
kubectl apply -f manifests/frontend-deployment.yaml
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

