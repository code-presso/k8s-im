# 06장 고급 스케줄링

## 고가용성 확보 - Pod 레벨

### Metrics server 설치

```bash
# Already install by kubespray
# helm install metrics-server bitnami/metrics-server

kubectl get pod -n kube-system
# NAME                   READY   STATUS    RESTARTS   AGE
# metrics-server-xxxxx   1/1     Running   0          2h
```

```bash
# 리소스 사용량을 모니터링할 Pod를 하나 생성합니다.
kubectl run mynginx --image nginx

# Pod별 리소스 사용량을 확인합니다.
kubectl top pod
# NAME        CPU(cores)   MEMORY(bytes)
# mynginx     0m           2Mi

# Node별 리소스 사용량을 확인합니다.
kubectl top node
# NAME    CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# node1   216m         12%    1905Mi          58%
# node2   182m         10%    1979Mi          60%
# node3   132m         6%     1745Mi          49%

kubectl delete pod mynginx
# pod/mynginx deleted
```

### 자동 확장할 Pod 생성

```php
# image: k8s.gcr.io/hpa-example
<?php
  $x = 0.0001;
  for ($i = 0; $i <= 1000000; $i++) {
    $x += sqrt($x);
  }
  echo "OK!";
?>
```

```yaml
# heavy-cal.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: heavy-cal
spec:
  selector:
    matchLabels:
      run: heavy-cal
  replicas: 1
  template:
    metadata:
      labels:
        run: heavy-cal
    spec:
      containers:
      - name: heavy-cal
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 300m
---
apiVersion: v1
kind: Service
metadata:
  name: heavy-cal
spec:
  ports:
  - port: 80
  selector:
    run: heavy-cal
```

```bash
kubectl apply -f heavy-cal.yaml
# deployment.apps/heavy-cal created
# service/heavy-cal created
```

### `hpa` 생성 - 선언형 명령

```yaml
# hpa.yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: heavy-cal
spec:
  maxReplicas: 50
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: heavy-cal
  targetCPUUtilizationPercentage: 50
```

```bash
# hpa 리소스 생성
kubectl apply -f hpa.yaml
# horizontalpodautoscaler.autoscaling/heavy-cal autoscaled
```

### 자동확장 테스트

```yaml
# heavy-load.yaml
apiVersion: v1
kind: Pod
metadata:
  name: heavy-load
spec:
  containers: 
  - name: busybox
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do wget -q -O- http://heavy-cal; done"]
```

```bash
kubectl apply -f heavy-load.yaml
# pod/heavy-load created

# watch문으로 heavy-cal를 계속 지켜보고 있으면 Pod의 개수가 증가하는 것을 확인할 수 있습니다.
watch kubectl top pod
# NAME                         CPU(cores)   MEMORY(bytes)
# heavy-load                   7m           1Mi
# heavy-cal-548855cf99-9s44l   140m         12Mi
# heavy-cal-548855cf99-lnbvm   122m         13Mi
# heavy-cal-548855cf99-lptbq   128m         13Mi
# heavy-cal-548855cf99-qpdng   89m          12Mi
# heavy-cal-548855cf99-tvgfn   137m         13Mi
# heavy-cal-548855cf99-x64mg   110m         12Mi
```

```bash
kubectl delete pod heavy-load
```

## 고가용성 확보 - Node 레벨

### AWS EKS Cluster AutoScaler 설정

```bash
NAME=k8s
REGION=ap-northeast-2

helm install autoscaler stable/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=$NAME,awsRegion=$REGION,sslCertPath=/etc/kubernetes/pki/ca.crt
  --version 7.3.4
```

### GCP GKE Cluster AutoScaler 설정

```bash
CLUSTER_NAME=k8s
REGION=us-central1-a

gcloud container clusters create $CLUSTER_NAME \
    --enable-autoscaling \
    --min-nodes=1 \
    --num-nodes=2 \
    --max-nodes=4 \
    --node-locations=$CLUSTER_NAME \
    --machine-type=n1-highcpu-8
```

### Cluster AutoScaling 활용

```bash
# 인위적으로 Pod 증가
kubectl scale deployment heavy-cal --replicas=50
# deployment.apps/heavy-cal scaled

# Pod 리스트
kubectl get pod
# NAME                        READY   STATUS    RESTARTS   AGE
# heavy-cal-548855cf99-x64m   1/1     Running   0          2s
# heavy-cal-548855cf99-dfx2   1/1     Running   0          2s
# heavy-cal-548855cf99-sf3x   1/1     Running   0          2s
# ....
# heavy-cal-548855cf99-a21t   0/1     Pending   0          2s
# heavy-cal-548855cf99-g8ib   0/1     Pending   0          2s
# heavy-cal-548855cf99-b754   0/1     Pending   0          2s
# ...

watch kubectl get node
# NAME             STATUS   ROLES    AGE   VERSION
# ip-172-31-42-5   Ready             13h   v1.18.6
# ip-172-31-44-9   Ready             1m    v1.18.6
# ....             Ready             1m    v1.18.6
# ....
```

### Clean up

```bash
kubectl delete hpa heavy-cal
kubectl delete deploy heavy-cal
kubectl delete svc heavy-cal
```

## `Taint & Toleration`

### Taint

```bash
# taint 방법
kubectl taint nodes $NODE_NAME <KEY>=<VALUE>:<EFFECT>
```

```bash
# project=A 라는 taint를 NoSchedule로 설정
kubectl taint node node1 project=A:NoSchedule
# node/node1 tainted

kubectl get node node1 -oyaml | grep -A 4 taints
#  taints:
#  - effect: NoSchedule
#    key: project
#    value: A
```

### Toleration

```yaml
# no-tolerate.yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-tolerate
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    kubernetes.io/hostname: node1
```

```bash
kubectl apply -f no-tolerate.yaml
# pod/no-tolerate created

kubectl get pod -o wide
# NAME            READY   STATUS    RESTARTS   AGE     IP             NODE   
# no-tolerate     0/1     Pending   0          11s     <none>         <none> 
```

```yaml
# tolerate.yaml
apiVersion: v1
kind: Pod
metadata:
  name: tolerate
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "project"
    value: "A"
    operator: "Equal"
    effect: "NoSchedule"
  nodeSelector:
    kubernetes.io/hostname: node1
```

```bash
kubectl apply -f tolerate.yaml
# pod/tolerate created

kubectl get pod -o wide
# NAME            READY   STATUS       RESTARTS   AGE
# no-tolerate     0/1     Pending      0          2m7s
# tolerate        1/1     Running      0          5s
```

```bash
# node1에 taint를 추가합니다.
# 이번에는 key만 존재하는 taint를 적용해 봅니다.
kubectl taint node node1 badsector=:NoSchedule
# node/node1 tainted

kubectl get node node1 -oyaml | grep -A 7 taints
#  taints:
#  - effect: NoSchedule
#    key: project
#    value: A
#  - effect: NoSchedule
#    key: badsector
```

```yaml
# badsector.yaml
apiVersion: v1
kind: Pod
metadata:
  name: badsector
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "project"
    value: "A"
    operator: "Equal"
    effect: "NoSchedule"
  - key: "badsector"
    operator: "Exists"
  nodeSelector:
    kubernetes.io/hostname: node1
```
 
```bash
kubectl apply -f badsector.yaml
# pod/badsector created

kubectl get pod -o wide
# NAME         READY   STATUS    RESTARTS   AGE    IP              NODE
# no-tolerate  0/1     Pending   0          2m7s   <none>          <none>
# tolerate     1/1     Running   0          2m     10.233.90.8     node1
# badsector    1/1     Running   0          16s    10.233.87.23    node1
```

```bash
# project taint 제거
kubectl taint node node1 project-
# badsector taint 제거
kubectl taint node node1 badsector-

kubectl get pod -o wide
# NAME         READY   STATUS              RESTARTS   AGE    IP              NODE
# no-tolerate  0/1     ContainerCreating   0          2m7s   <none>          <none>
# tolerate     1/1     Running             0          2m     10.233.90.8     node1
# badsector    1/1     Running             0          16s    10.233.87.23    node1
```

### Clean up

```bash
kubectl delete pod no-tolerate tolerate badsector
```


## `Affinity & AntiAffinity`

### `NodeAffinity`

```yaml
# node-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity
spec:
  containers:
  - name: nginx
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

```bash
kubectl label node node3 disktype=ssd

kubectl apply -f node-affinity.yaml
# pod/node-affinity created

kubectl get pods node-affinity -o wide
# NAME           READY   STATUS    RESTARTS  AGE   IP          NODE   ..
# node-affinity  1/1     Running   0         19s   10.42.0.8   node3  ..
```

### `PodAffinity`

```yaml
# pod-affinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-affinity
spec:
  selector:
    matchLabels:
      app: affinity
  replicas: 2
  template:
    metadata:
      labels:
        app: affinity
    spec:
      containers:
      - name: nginx
        image: nginx
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - affinity
            topologyKey: "kubernetes.io/hostname"
```

```bash
kubectl apply -f pod-affinity.yaml
# deployment.apps/pod-affinity created

kubectl get pod -o wide
# NAME              READY  STATUS    RESTARTS  AGE   IP             NODE
# pod-affinity-xxx  1/1    Running   0         11m   10.42.0.165    node1
# pod-affinity-xxx  1/1    Running   0         11m   10.42.0.166    node1
```

### `PodAntiAffinity`

```yaml
# pod-antiaffinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-antiaffinity
spec:
  selector:
    matchLabels:
      app: antiaffinity
  replicas: 2
  template:
    metadata:
      labels:
        app: antiaffinity
    spec:
      containers:
      - name: nginx
        image: nginx
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - antiaffinity
            topologyKey: "kubernetes.io/hostname"
```

```bash
kubectl apply -f pod-antiaffinity.yaml
# deployment.apps/pod-antiaffinity created

kubectl get pod -o wide
# NAME                 READY  STATUS    RESTARTS AGE  IP           NODE
# pod-antiaffinity-xxx 1/1    Running   0        10s  10.42.0.168  node1
# pod-antiaffinity-xxx 1/1    Running   0        11s  10.42.0.167  node2
```

### `PodAffinity`와 `PodAntiAffinity` 활용법

#### cache 서버 설정

```yaml
# redis-cache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 2
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        # cache 서버끼리 멀리 스케줄링
        # app=store 라벨을 가진 Pod끼리 멀리 스케줄링
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis
```

#### web 서버 설정

```yaml
# web-server.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 2
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        # web 서버끼리 멀리 스케줄링
        # app=web-store 라벨을 가진 Pod끼리 멀리 스케줄링
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        # web-cache 서버끼리 가까이 스케줄링 
        # app=store 라벨을 가진 Pod끼리 가까이 스케줄링
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx
```

```bash
kubectl apply -f redis-cache.yaml -f web-server.yaml
# deployment.app/redis-cache created
# deployment.app/web-server created


kubectl get pod -owide
# NAME             READY  STATUS    RESTARTS  AGE     IP            NODE
# redis-cache-xxx  1/1    Running   0         10s     10.42.0.151   node3
# redis-cache-xxx  1/1    Running   0         10s     10.42.0.152   node2
# web-server-xxxx  1/1    Running   0         11s     10.42.0.153   node3
# web-server-xxxx  1/1    Running   0         11s     10.42.0.154   node2
```

### Clean up

```bash
kubectl delete -f node-affinity.yaml -f pod-affinity.yaml -f pod-antiaffinity.yaml -f redis-cache.yaml -f web-server.yaml
```
