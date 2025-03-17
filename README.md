# Deploying a Kubernetes Voting App on Azure Kubernetes Service (AKS)

This guide provides step-by-step instructions to deploy a cloud-native voting application on Azure Kubernetes Service (AKS). The application consists of a React-based frontend, a Go-based backend API, and a MongoDB database, demonstrating a typical microservices architecture.

## Prerequisites

1.  **Azure CLI**: Install and configure Azure CLI.
    * Installation (PowerShell): `Start-Process "https://aka.ms/installazurecliwindows"`
    * Verification: `az --version`
    * Login: `az login --allow-no-subscriptions`
2.  **kubectl**: Install Kubernetes CLI.
    * Installation (Linux): `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"; chmod +x kubectl; sudo mv kubectl /usr/local/bin/`
    * Verification: `kubectl version --client`
3.  **Helm**: Install Helm for package management.
4.  **Azure Subscription**: Ensure you have an active Azure subscription.
5.  **Git**: Install Git for cloning the repository.

## Deployment Steps

1.  **Create an Azure Resource Group:**
    ```sh
    az group create --name AKSResourceGroup --location eastus
    ```
2.  **Register the ContainerService Provider:**
    ```sh
    az provider register --namespace Microsoft.ContainerService
    az provider show --namespace Microsoft.ContainerService --query "registrationState"
    ```
3.  **Create an AKS Cluster:**
    ```sh
    az aks create --resource-group AKSResourceGroup --name MyAKSCluster --node-count 1 --node-vm-size Standard_B2s --generate-ssh-keys --tier free
    ```
4.  **Configure kubectl to Connect to AKS:**
    ```sh
    az aks get-credentials --resource-group AKSResourceGroup --name MyAKSCluster
    ```
5.  **Verify the Cluster:**
    ```sh
    kubectl get nodes
    ```
6.  **Clone the GitHub Repository:**
    ```sh
    git clone https://github.com/chidhvilaskacham/k8s-voteapp.git
    cd k8s-voteapp
    ```
7.  **Create a Kubernetes Namespace:**
    ```sh
    kubectl create ns k8s-voteapp
    kubectl config set-context --current --namespace k8s-voteapp
    ```
8.  **Deploy MongoDB:**
    * Create MongoDB Deployment:
        ```sh
        kubectl apply -f mongo-deployment.yaml
        ```
    * Verify MongoDB DNS Resolution:
        ```sh
        kubectl run --rm utils -it --image praqma/network-multitool -- bash
        for i in {0..2}; do nslookup mongo-$i.mongo; done
        exit
        ```
    * Initialize MongoDB Replica Set:
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
    * Verify MongoDB Replica Set:
        ```sh
        kubectl exec -it mongo-0 -- mongo --eval "rs.status()" | grep "PRIMARY\|SECONDARY"
        ```
9.  **Load Data into MongoDB:**
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
10. **Deploy the API:**
    ```sh
    kubectl apply -f backend-deployment.yaml
    ```
11. **Get API Endpoint and Test:**
    ```sh
    REACT_APP_BACKEND_URL=$(kubectl get svc go-backend-service -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
    curl $REACT_APP_BACKEND_URL/ok
    curl -s $REACT_APP_BACKEND_URL/tools | jq .
    ```
12. **Deploy the Frontend:**
    ```sh
    kubectl apply -f frontend-deployment.yaml
    ```
13. **Get Frontend URL:**
    ```sh
    FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc react-frontend-service -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
    echo http://$FRONTEND_ELB_PUBLIC_FQDN
    ```
14. **Test the Application:**
    * Open the frontend URL in a browser and interact with the voting application.
15. **Query MongoDB for Updated Votes:**
    ```sh
    kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
    ```
16. **Cleanup Resources:**
    ```sh
    az group delete --name AKSResourceGroup --yes --no-wait
    ```
