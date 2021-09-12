# 02장 Kubespray

## kubespray setup 하기

kubespray: [https://github.com/kubernetes-sigs/kubespray](https://github.com/kubernetes-sigs/kubespray)

```bash
MASTER='x.x.x.x'
WORKER1='x.x.x.x'
WORKER2='x.x.x.x'

git clone https://github.com/kubernetes-sigs/kubespray.git -b v2.15.1
cd kubespray

wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh

# Install dependencies from ``requirements.txt``
pip install -r requirements.txt
# pip install --ignore-installed ruamel.yaml

# Copy ``inventory/sample`` as ``inventory/mycluster``
cp -rfp inventory/sample inventory/mycluster

# Update Ansible inventory file with inventory builder
declare -a IPS=($MASTER $WORKER1 $WORKER2)
echo ${IPS[@]}
CONFIG_FILE=inventory/mycluster/hosts.yaml python contrib/inventory_builder/inventory.py ${IPS[@]}

# Create public key
ssh-keygen -t rsa

# Deploy to master, worker
scp ~/.ssh/id_rsa.pub $MASTER:~/.ssh/authorized_keys2
scp ~/.ssh/id_rsa.pub $WORKER1:~/.ssh/authorized_keys2
scp ~/.ssh/id_rsa.pub $WORKER2:~/.ssh/authorized_keys2

# ping
ansible all -i inventory/mycluster/hosts.yaml  -m ping
```

## kubespray configuration

`inventory/mycluster/group_vars/`

- `all/all.yml`: control plane 관련 설정값
- `all/docker.yml`: 도커 설정값
- `k8s_cluster/k8s-cluster.yml`: 클러스터 전반 (버전, 네트워크, authN, 시스템 리소스 사용량)
- `k8s_cluster/k8s-net-calico.yml`: 네트워크 설정 (Pod, Service cidr, NAT)
- `k8s_cluster/addons.yml`: 필수 컴포넌트는 아니나 자주 사용하는 컴포넌트 리스트


- `inventory/mycluster/group_vars/k8s-cluster/addons.yml`
    - `metrics_server_enabled: false` --> `true`

## Cluster Deploy

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get node -owide

# RBAC for local-path-provisioner
kubectl create clusterrolebinding localpath-admin --clusterrole cluster-admin --serviceaccount=local-path-storage:local-path-provisioner-service-account
```

## Cluster Remove Node

```bash
NODE_NAME='node3'
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root -e node=$NODE_NAME remove-node.yml

kubectl get node -owide
```


## Cluster Add Node

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root scale.yml

kubectl get node -owide
```

## Cluster Shutdown

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root reset.yml
```
