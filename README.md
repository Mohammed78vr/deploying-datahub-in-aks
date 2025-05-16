# Deploying DataHub on Azure Kubernetes Service (AKS)

## Overview


This repository documents the deployment of DataHub, an open-source metadata platform, on Azure Kubernetes Service (AKS). While following the [official DataHub deployment guide for Azure](https://docs.datahub.com/docs/deploy/azure), I encountered some challenges that required troubleshooting and adjustments to ensure a successful setup.

Through this wiki, I aim to share my deployment process, highlight key configurations, and provide fixes for common errors to help others set up DataHub efficiently in AKS.

## Prerequisites

Before deploying DataHub, ensure you have the following:

* **Azure Subscription** – Required for provisioning resources on Azure.

* **kubectl Installed** – To interact with the AKS cluster.

* **Helm Installed** – For managing Kubernetes packages and deployments.

* **Azure CLI Installed** – To manage Azure services and configurations from the command line.

## Architecture Overview


![cross-bootcamp-project](https://github.com/user-attachments/assets/4f56703a-d183-4b2c-bbbf-af2363dbf541)


The deployment of **DataHub** in **Azure Kubernetes Service (AKS)** relies on several core Azure networking and compute services to ensure scalability, security, and efficient data ingestion.



**Key Components**


* **Azure Kubernetes Service (AKS)** – Hosts and orchestrates the deployment of DataHub services.

* **Application Gateway** – Acts as a Layer 7 load balancer, managing inbound traffic to AKS and enforcing security policies.

* **Virtual Network (VNet)** – Provides network isolation, ensuring secure communication between DataHub components and external services.

* **Subnet** – A segment within the VNet used to allocate resources such as AKS nodes and Application Gateway.

* **AWS RDS MySQL** – Used primarily for data ingestion, enabling DataHub to interact with external metadata sources.

**Flow of Operations**


* **User Requests** – External traffic is directed to Application Gateway, which filters and forwards it to AKS.

* **Networking & Routing** – Requests pass through the VNet and Subnet, ensuring secure, controlled access.

* **DataHub Services** – Inside AKS, core services (GMS, frontend, ingestion pipelines) process metadata and user interactions.

* **Data Ingestion** – DataHub ingests metadata from AWS RDS MySQL, utilizing it as a source for cataloging and indexing.

## Deployment Walkthrough

This section provides a detailed guide on deploying DataHub in Azure Kubernetes Service (AKS) using Helm and Azure CLI.

### Step 1: Set Up Your AKS Cluster

**1. Log in to Azure using Azure CLI:**

```bash
az login
```

**2. Create a resource group**

```bash
az group create --name myResourceGroup --location eastus
```

**Output result:**
```bash
{
  "id": "/subscriptions/<guid>/resourceGroups/myResourceGroup",
  "location": "eastus",
  "managedBy": null,
  "name": "myResourceGroup",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```

**3. Deploy an AKS cluster:**

```bash
az aks create -g myResourceGroup -n myAKSCluster --enable-managed-identity --node-count 3 --enable-addons monitoring --generate-ssh-keys
```

> Note: you may get an error because the size of the vm is not supported in your region. so you need to specify it in order avoid the error like this

```bash
az aks create -g DataHubAksRg -n DatahubAKSCluster --enable-managed-identity --node-count 2 --enable-addons monitoring --generate-ssh-keys --node-vm-size Standard_B2s --location canadacentral
```


**4. Connect to the AKS cluster:**
```bash
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```
> Note: Because I was running the commands in the WSL I got an error and that's because the .kubectl config file was stored in the windows file. So, make sure that the .kubectl config file is present in you working machine and I run this command `cp /mnt/c/Users/<user>/.kube/config ~/.kube/config`

**5. Verify the connection to your cluster:**
```bash
kubectl get nodes
```

You should get results like below. Make sure node status is Ready.

```bash
aks-nodepool1-10606016-vmss000000   Ready    <none>   3m19s   v1.31.7
aks-nodepool1-10606016-vmss000001   Ready    <none>   3m20s   v1.31.7
aks-nodepool1-10606016-vmss000002   Ready    <none>   3m16s   v1.31.7
```

### Step 2: Deploying DataHub in the AKS 

**1. Create kubernetes secrets that contain MySQL and Neo4j passwords:**
```bash
kubectl create secret generic mysql-secrets --from-literal=mysql-root-password=datahub
kubectl create secret generic neo4j-secrets --from-literal=neo4j-password=datahub
```

**output:**
```bash
secret/mysql-secrets created
secret/neo4j-secrets created
```


**2. Add datahub helm repo by running the following:**
```bash
helm repo add datahub https://helm.datahubproject.io/
```


**3. Deploy the dependencies**

```bash
helm install prerequisites datahub/datahub-prerequisites
```

**output**
```
NAME: prerequisites
LAST DEPLOYED: Thu May 15 09:30:16 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
```

**4. Verify that the dependencies are running:**

```bash
NAME                           READY   STATUS    RESTARTS   AGE
elasticsearch-master-0         0/1     Running   0          35s
prerequisites-kafka-broker-0   0/1     Running   0          35s
prerequisites-mysql-0          0/1     Running   0          35s
prerequisites-zookeeper-0      0/1     Running   0          35s
```

**5. Deploy Datahub by running the following**

```bash
helm install datahub datahub/datahub
```

> Note: It will take time to finish.

**Output:**

```bash
NAME: datahub
LAST DEPLOYED: Thu May 15 09:31:09 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

**6. Verify whether all the datahub pods are running:**

```bash
kubectl get pods
```

**Output:**

```bash
NAME                                             READY   STATUS      RESTARTS   AGE
datahub-acryl-datahub-actions-5fb7575cb8-gntdk   1/1     Running     0          2m18s
datahub-datahub-frontend-5d5c75dbdf-nsh57        1/1     Running     0          2m18s
datahub-datahub-gms-6fb76fb844-5tdkz             0/1     Running     0          2m18s
datahub-elasticsearch-setup-job-b5dtb            0/1     Completed   0          7m5s
datahub-kafka-setup-job-vz5jk                    0/1     Completed   0          6m27s
datahub-mysql-setup-job-qpr5h                    0/1     Completed   0          4m35s
datahub-system-update-h8nbg                      0/1     Completed   0          4m27s
datahub-system-update-nonblk-2z44r               0/1     Completed   0          2m15s
elasticsearch-master-0                           1/1     Running     0          7m55s
prerequisites-kafka-broker-0                     1/1     Running     0          7m55s
prerequisites-mysql-0                            1/1     Running     0          7m55s
prerequisites-zookeeper-0                        1/1     Running     0          7m55s
```

**7. Expose the datahub-frontend to access the tool in localhost**

```bash
kubectl port-forward <datahub-frontend pod name> 9002:9002
```

you can use `http://localhost:9002` to access the tools.


### Step 3: Deploying Application Gateway and connecting it to the AKS

**1. In the Azure portal search for Application gateway and click it to create new one.**


fill the requirements and make sure that you have subnet in the same Vnet as the AKS. Otherwise, we cannot enable AGIC.

![image](https://github.com/user-attachments/assets/aa346bba-2542-4a1c-a1c0-7ba336bb3f5d)


**2. Create a new public IP address.**

![image](https://github.com/user-attachments/assets/335a82d2-e653-4e13-9ece-fb05a5e26411)


**3. Create an empty backend pool.**

![image](https://github.com/user-attachments/assets/bb0fc04b-d8e1-4715-8584-cd5477e8e712)

**4. Create an routing rule.**

for the Listener:

![image](https://github.com/user-attachments/assets/1dde7031-f558-4127-8fc3-4b27b2a811ae)

For the Backend settings create one with the following specification

![image](https://github.com/user-attachments/assets/02d1de26-888a-448c-98f0-ff1d6a5b35d2)

**5. Click next and then create.**

**6. enabling the Application Gateway Ingress Controller (AGIC)**
> Note: Replace the `$appgwId` with the recourse ID of the application gateway.
to get the recourse ID go to the application gateway -> settings -> properties

```bash
az aks enable-addons -n myAKSCluster -g myResourceGroup -a ingress-appgw --appgw-id $appgwId
```

**7. Deploy the Ingress on the Frontend Pod**

In order to use the ingress controller to expose frontend pod, we need to update the datahub-frontend section of the `values.yaml` file that was used to deploy DataHub

copy and paste this configuration in the `values.yaml` file

```bash
datahub-frontend:
  enabled: true
  image:
    repository: acryldata/datahub-frontend-react
    # tag: "v0.10.0 # defaults to .global.datahub.version

  # Set up ingress to expose react front-end
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: azure/application-gateway
      appgw.ingress.kubernetes.io/backend-protocol: "http"

    hosts:
    - paths:
      - /*
  defaultUserCredentials: {}
```

**8. Apply the updates:**

```bash
helm upgrade datahub datahub/datahub -f <yaml-file>
```

**9. Verify that the ingress was created correctly**

```bash
kubectl get ingress
```

**Outputs**

```bash
NAME                       CLASS    HOSTS   ADDRESS         PORTS   AGE
datahub-datahub-frontend   <none>   *       20.185.212.31   80      131m
```

You can use the public IP address of the application gateway to access the datahub.

## Issues I encounter while following the documentation.

The Issue I faced is that in the [Datahub deployment](https://docs.datahub.com/docs/deploy/azure) is that it makes a second Vnet to deploy the application inside it and then making Vnet peering between AKS Vnet and Application gateway Vnet which is wrong in my case because I cannot enable AGIC while the application gateway in different Vnet. To solve this issue I deploy the application gateway in the same Vnet as the AKS.

## Resources
Below are useful links and references for deploying DataHub on Azure Kubernetes Service (AKS):

**[Official DataHub Documentation](https://docs.datahub.com/docs/deploy/azure)**

**[Azure Kubernetes Service (AKS) Docs](https://learn.microsoft.com/en-us/azure/aks)**

**[Helm Documentation](https://helm.sh/docs/)**

**[Azure CLI Reference](https://learn.microsoft.com/en-us/cli/azure)**

**[Kubernetes Documentation](https://kubernetes.io/docs/)**
