# Real time monitoring of Prometheus and Grafana Dashboard on EKS Cluster using Helm Chart

Setting up the Infra on AWS with EKS Kubernetes Cluster 

![image](https://github.com/Pavan-1997/EKS_Prometheus_Helm_Grafana/assets/32020205/6268c034-9983-4db2-9b86-537dec9dbab4)

---
## Step 1 - Setup EC2 Instance

- Instance Type as t2.medium
- AMIs as Ubuntu


### Step 1.1 – Attach the IAM role having full access

## Step 2 - Install AWS CLI and Configure
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o"awscliv2.zip"

sudo apt install unzip

unzip awscliv2.zip

sudo ./aws/install
```

## Step 3 - Install and Setup Kubectl
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin

kubectl version
```

## Step 4 - Install and Setup eksctl
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp 

sudo mv /tmp/eksctl /usr/local/bin

eksctl version
```

## Step 5 - Install Helm chart
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh

helm version
```

## Step 6 - Creating an Amazon EKS cluster using eksctl
```
eksctl create cluster --name eks2 --version 1.24 --region us-east-1 --nodegroup-name worker-nodes --node-type t2.large --nodes 2 --nodes-min 2 --nodes-max 3
```

## Step 7 - Installing the Kubernetes Metrics Server
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Step 7.1 - Verify that the metrics-server deployment is running the desired number of pods with the following command
```
kubectl get deployment metrics-server -n kube-system
```

## Step 8 - Install Prometheus

- Now install the Prometheus using the helm chart.

- Add Prometheus helm chart repository
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

## Step 8.1 - Update helm chart repository
```
helm repo update
```
```
helm repo list
```

### Step 8.2 - Create prometheus namespace
```
kubectl create namespace prometheus
```

### Step 8.3 - Install Prometheus
```
helm install prometheus prometheus-community/prometheus \--namespace prometheus \--set alertmanager.persistentVolume.storageClass="gp2" \--set server.persistentVolume.storageClass="gp2"
```

## Step 9 - Create IAM OIDC Provider

- Your cluster has an OpenID Connect (OIDC) issuer URL associated with it. 

- To use AWS Identity and Access Management (IAM) roles for service accounts, an IAM OIDC provider must exist for your cluster's OIDC issuer URL.
```
oidc_id=$(aws eks describe-cluster --name eks2 --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```
```
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```
```
eksctl utils associate-iam-oidc-provider --cluster eks2 --approve --region us-east-1
```

## Step 10 - Create iamserviceaccount with role
```
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster eks2 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
--region us-east-1
```

### Step 10.1 - Then attach ROLE to eks by running the following command

- Enter your account ID and cluster name.
```
eksctl create addon --name aws-ebs-csi-driver --cluster eks2 --service-account-role-arn arn:aws:iam::716612577713:role/AmazonEKS_EBS_CSI_DriverRole --force --region us-east-1
```

### Step 10.2 - Verify pods in Prometheus
```
kubectl get pods -n prometheus
```

### Step 10.3 - View the Prometheus dashboard by forwarding the deployment ports
```
kubectl port-forward deployment/prometheus-server 9090:9090 -n prometheus
```
 
### Step 10.4 - Open different browser and connect to your EC2 instance and run the below command
```
curl localhost:9090/graph
```
 
## Step 11 - Install Grafana
```
helm repo add grafana https://grafana.github.io/helm-charts
```
```
helm repo update
```

### Step 11.1 - Now we need to create a Prometheus data source so that Grafana can access the Kubernetes metrics
```

#Create a yaml file prometheus-datasource.yaml and save the following data source configuration into it

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.prometheus.svc.cluster.local
      access: proxy
      isDefault: true

```                                                                                                                                                                               
### Step 11.2 - Create a namespace Grafana
```
kubectl create namespace grafana
```

### Step 11.3 - Install the Grafana
```
#This below command will create the Grafana service with an external load balancer to get the public view

helm install grafana grafana/grafana \
  --namespace grafana \
  --set persistence.storageClassName="gp2" \
  --set persistence.enabled=true \
  --set adminPassword='EKS!sAWSome' \
  --values prometheus-datasource.yaml \
  --set service.type=LoadBalancer
```

### Step 11.4 - Verify the Grafana installation by using the following kubectl command
```
kubectl get pods -n grafana
```
```
kubectl get service -n grafana
```

### Step 11.5 - Copy the EXTERNAL-IP and paste in the browser

The password you mentioned as EKS!sAWSome while creating Grafana Step 10.3

### Step 11.6 - Import Grafana dashboard from Grafana Labs

For the custom Grafana Dashboard, we are going to use the open-source grafana dashboard. For this session, I am going to import a dashboard 6417

Load and select the source as Prometheus

## Step 12 - Visualise the Java application

## Step 13 - Deploy a Java application and monitor it on Grafana
```
git clone https://github.com/Pavan-1997/Java_Kubernetes_Deployment.git
```
```
cd /Java_Kubernetes_Deployment/Kubernetes
```
```
kubectl apply -f shopfront-service.yaml
```
```
kubectl get deployment
```
```
kubectl get pods
```
```
kubectl logs shopfront-7868468c56-4r2kk -c shopfront
```

## Step 14 - Clean Up
```
eksctl delete cluster --name eks2 --region us-east-1
```
