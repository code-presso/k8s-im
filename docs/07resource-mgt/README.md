# 07장 리소스 관리

## 리소스 관리

### LimitRange

```bash
kubectl run mynginx --image nginx

kubectl get pod mynginx -oyaml | grep resources
# resources: {}
```

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
spec:
  limits:
  - default:
      cpu: 400m
      memory: 512Mi
    defaultRequest:
      cpu: 300m
      memory: 256Mi
    max:
      cpu: 600m
      memory: 600Mi
    min:
      cpu: 200m
      memory: 200Mi
    type: Container
```

```bash
kubectl apply -f limit-range.yaml
# limitrange/limit-range created

kubectl run nginx-lr --image nginx
# pod/nginx-lr created

kubectl get pod nginx-lr -oyaml | grep -A 6 resources
#    resources:
#      limits:
#        cpu: 400m
#        memory: 512Mi
#      requests:
#        cpu: 300m
#        memory: 256Mi
```

```yaml
# pod-exceed.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-exceed
spec:
  containers:
  - image: nginx
    name: nginx
    resources:
      limits:
        cpu: "700m"
        memory: "700Mi"
      requests:
        cpu: "300m"
        memory: "256Mi"
```

```bash
kubectl apply -f pod-exceed.yaml
# Error from server (Forbidden): error when creating "pod-exceed.yaml": pods "pod-exceed" is forbidden: [maximum cpu usage per Container is 600m, but limit is 700m, maximum memory usage per Container is 600Mi, but limit is 700Mi]
```

### Clean up

```bash
kubectl delete limitrange limit-range
kubectl delete pod nginx-lr mynginx
```

### ResourceQuota

```yaml
# res-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: res-quota
spec:
  hard:
    limits.cpu: 700m
    limits.memory: 800Mi
    requests.cpu: 500m
    requests.memory: 700Mi
```

```bash
# ResourceQuota 생성
kubectl apply -f res-quota.yaml
# resourcequota/res-quota created 

# Pod 생성 limit CPU 600m
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rq-1
spec:
  containers:
  - image: nginx
    name: nginx
    resources:
      limits:
        cpu: "600m"
        memory: "600Mi"
      requests:
        cpu: "300m"
        memory: "300Mi"
EOF
# pod/rq-1 created
```

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rq-2
spec:
  containers:
  - image: nginx
    name: nginx
    resources:
      limits:
        cpu: "600m"
        memory: "600Mi"
      requests:
        cpu: "300m"
        memory: "300Mi"
EOF
# Error from server (Forbidden): error when creating "STDIN": pods "rq-2" is forbidden: exceeded quota: res-quota, requested: limits.cpu=600m,limits.memory=600Mi,requests.cpu=300m, used: limits.cpu=600m,limits.memory=600Mi,requests.cpu=300m, limited: limits.cpu=700m,limits.memory=800Mi,requests.cpu=500m
```

### Clean up

```bash
kubectl delete resourcequota res-quota
kubectl delete pod rq-1
```

## 노드 관리

### Cordon

```bash
kubectl cordon <NODE>
```

```bash
# 먼저 worker의 상태를 확인합니다.
kubectl get node node2 -oyaml | grep spec -A 5
# spec:
#   podCIDR: 10.42.0.0/24
#   podCIDRs:
#   - 10.42.0.0/24
# status:

# worker를 cordon시킵니다.
kubectl cordon node2
# node/node2 cordoned

kubectl get node node2 -oyaml | grep taints -A 5
#   taints:
#   - effect: NoSchedule
#     key: node.kubernetes.io/unschedulable
#     timeAdded: "2021-06-13T09:14:19Z"
#   unschedulable: true
# status:

# worker의 상태를 확인합니다.
kubectl get node
# NAME    STATUS                     ROLES    AGE   VERSION
# node1   Ready                      master   21h   v1.19.9
# node2   Ready,SchedulingDisabled   <none>   21h   v1.19.9
# node3   Ready                      <none>   20h   v1.19.9
```

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs
spec:
  replicas: 5
  selector:
    matchLabels:
      run: rs
  template:
    metadata:
      labels:
        run: rs
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
```

```bash
# No Pod in node2
kubectl get pod -o wide
# NAME                               READY   STATUS              RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
# nfs-provisioner-78dc99f5b7-kf8hj   1/1     Running             0          3h45m   10.233.92.32   node3   <none>           <none>
# rs-7p7n2                           0/1     ContainerCreating   0          6s      <none>         node1   <none>           <none>
# rs-82jq6                           1/1     Running             0          6s      10.233.90.13   node1   <none>           <none>
# rs-8prv8                           1/1     Running             0          6s      10.233.92.53   node3   <none>           <none>
# rs-cs58q                           0/1     ContainerCreating   0          6s      <none>         node3   <none>           <none>
# rs-wb597                           0/1     ContainerCreating   0          6s      <none>         node3   <none>           <none>
```

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-node2
spec:
  containers:
  - image: nginx
    name: nginx
  nodeSelector:
    kubernetes.io/hostname: node2
EOF
# pod/pod-node2 created

kubectl get pod -owide
# NAME        READY  STATUS    RESTARTS   AGE     IP       NODE    ... 
# ...
# pod-node2   0/1    Pending   0          70s     <none>   <none>  ...
```

### Uncordon

```bash
kubectl uncordon node2
# node/node2 uncordoned

kubectl get node node2 -oyaml | grep spec -A 10
# spec:
#   podCIDR: 10.42.1.0/24
#   podCIDRs:
#   - 10.42.1.0/24
# status:
#   addresses:
#   - address: 172.31.16.173
#     type: InternalIP
#   - address: node2
#     type: Hostname

kubectl get node
# NAME    STATUS   ROLES    AGE   VERSION
# node1   Ready    master   21h   v1.19.9
# node2   Ready    <none>   21h   v1.19.9
# node3   Ready    <none>   20h   v1.19.9


kubectl get pod pod-node2 -owide
# NAME        READY   STATUS    RESTARTS   AGE   IP       NODE     ...
# pod-node2   1/1     Running   0          70s   <none>   node2   ...

kubectl delete pod pod-node2
# pod/pod-node2 deleted

kubectl delete rs rs
```

### Drain

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
# deployment.apps/nginx created

kubectl get pod -o wide
# NAME               READY  STATUS    RESTARTS  AGE  IP           NODE
# nginx-7ff78b8-xxx  1/1    Running   0         42s  10.42.0.25   node1
# nginx-7ff78b8-xxx  1/1    Running   0         42s  10.42.1.2    node2
# nginx-7ff78b8-xxx  1/1    Running   0         42s  10.42.4.62   node3
```

```bash
# 모든 노드에 존재하는 DaemonSet은 무시합니다.
kubectl drain node3  --ignore-daemonsets
# node/node3 cordoned
# evicting pod "nginx-xxx"
# ...

# nginx Pod가 어떻게 동작하는지 확인합니다.
watch kubectl get pod -owide

 
kubectl get node node3 -oyaml | grep taints -A 5
#   taints:
#   - effect: NoSchedule
#     key: node.kubernetes.io/unschedulable
#     timeAdded: "2020-04-04T15:37:25Z"
#   unschedulable: true
# status:

kubectl get node
# NAME    STATUS                     ROLES    AGE   VERSION
# node1   Ready                      master   21h   v1.19.9
# node2   Ready                      <none>   21h   v1.19.9
# node3   Ready,SchedulingDisabled   <none>   20h   v1.19.9
```

```bash
kubectl uncordon node3
# node/node3 uncordoned
```

### Clean up

```bash
kubectl delete deploy nginx
```
