# React Frontend, Golang Backend & MongoDB on AKS

This repository contains Kubernetes configurations to deploy a **React frontend, Golang backend, and MongoDB** on **Azure Kubernetes Service (AKS)** inside the `k8s-voteapp` namespace with **public-facing Load Balancers**.

## **📌 Prerequisites**
Ensure you have the following installed on your machine:
- [Docker](https://www.docker.com/get-started)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [AKS Cluster](https://learn.microsoft.com/en-us/azure/aks/kubernetes-walkthrough)

---

## **🚀 Steps to Deploy**

### **1️⃣ Create the Namespace**
```sh
kubectl create namespace k8s-voteapp
```

### **2️⃣ Deploy MongoDB**
```sh
kubectl apply -f mongo-deployment.yaml
```
This will deploy MongoDB with a **ClusterIP service**.

### **3️⃣ Deploy Golang Backend**
```sh
kubectl apply -f backend-deployment.yaml
```
- The backend is exposed via a **public LoadBalancer**.
- It connects to MongoDB using: `mongodb://admin:password@mongo-service:27017`.

### **4️⃣ Deploy React Frontend**
```sh
kubectl apply -f frontend-deployment.yaml
```
- The frontend is exposed via a **public LoadBalancer**.
- It uses an environment variable to connect to the backend API.

---

## **📡 Verify Deployment**
### **Check Pods & Deployments**
```sh
kubectl get pods -n k8s-voteapp
kubectl get deployments -n k8s-voteapp
```
### **Check Services (External IPs)**
```sh
kubectl get services -n k8s-voteapp
```
Example output:
```
NAME                    TYPE           EXTERNAL-IP       PORT(S)
go-backend-service      LoadBalancer   52.165.32.45      8080:30201/TCP
react-frontend-service  LoadBalancer   52.150.28.12      80:30987/TCP
```

🔹 **Frontend Access:** `http://52.150.28.12`
🔹 **Backend Access:** `http://52.165.32.45:8080`

---

## **🔄 Updating Images**
If you make changes to the frontend/backend, rebuild & push images:
```sh
docker build -t your-docker-hub-username/my-react-app .
docker push your-docker-hub-username/my-react-app
kubectl rollout restart deployment react-frontend -n k8s-voteapp
```
```sh
docker build -t your-docker-hub-username/my-go-app .
docker push your-docker-hub-username/my-go-app
kubectl rollout restart deployment go-backend -n k8s-voteapp
```

---

## **🗑 Cleanup**
To delete the deployments:
```sh
kubectl delete namespace k8s-voteapp
```
To delete the AKS cluster:
```sh
az aks delete --name myAKSCluster --resource-group myResourceGroup --yes --no-wait
```
---

## **🎉 Congratulations!**  
You have successfully deployed a **React-Golang-MongoDB app** on **Azure Kubernetes Service (AKS)**! 🚀


