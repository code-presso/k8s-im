# [Bonus] Cloud Platforms


## EKS


### Create AdminRole


### Create Bastion Server

```bash
# install jq
# sudo apt-get update && \
#     sudo apt-get install -y jq apt-transport-https \
#     ca-certificates \
#     curl \
#     gnupg-agent \
#     software-properties-common 


# 클러스터 이름과 리전을 설정합니다.
CLUSTER_NAME=codepresso
REGION_NAME=ap-northeast-2

# installing eksctl
curl --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | \
    tar xz -C /tmp && \
    sudo mv /tmp/eksctl /usr/local/bin

# installing aws-iam-authenticator
curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/aws-iam-authenticator && \
    chmod +x ./aws-iam-authenticator && \
    sudo mv aws-iam-authenticator /usr/local/bin
    
   
# installing kubectl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && \
    echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list && \
    sudo apt-get update && \
    sudo apt-get install -y kubectl


# Create k8s cluster
eksctl create cluster --name $CLUSTER_NAME \
                      --nodes-min 1 \
                      --nodes 3 \
                      --nodes-max 5 \
                      --asg-access --region $REGION_NAME \
                      --version=1.20

# 클러스터 확인
kubectl get node -L role
```


## GKE

```bash
CLUSTER_NAME=codepresso
REGION_NAME=asia-northeast3-a
gcloud components update

gcloud config set compute/zone $REGION_NAME

gcloud container clusters create $CLUSTER_NAME \
    --num-nodes=1 \
    --machine-type=n1-standard-4 \
    --node-locations=$REGION_NAME \
    --enable-autoscaling \
    --min-nodes=1 \
    --num-nodes=3 \
    --max-nodes=5
```

## AKS

```bash
CLUSTER_NAME=codepresso
RESOURCE_GROUP=myrg
REGION_NAME=koreacentral

az group create \
    --name $RESOURCE_GROUP \
    --location $REGION_NAME

az aks create \
	--resource-group $RESOURCE_GROUP \
	--name $CLUSTER_NAME \
	--load-balancer-sku standard \
	--location $REGION_NAME \
	--kubernetes-version 1.21.1 \
	--network-plugin azure \
	--generate-ssh-keys \
    --enable-cluster-autoscaler true \
    --min-count 1 \
    --node-count 3 \
    --max-count 5

az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME
```