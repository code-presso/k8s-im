# 08장 접근제어

### HTTP Authentication

```bash
kubectl create sa mysa

kubectl get sa mysa -oyaml
# apiVersion: v1
# kind: ServiceAccount
# metadata:
#   creationTimestamp: "2021-06-12T11:56:24Z"
#   name: mysa
#   namespace: default
#   resourceVersion: "366"
#   selfLink: /api/v1/namespaces/default/serviceaccounts/mysa
#   uid: 26a257cc-8921-4e38-81a0-be5b2f47abe5
# secrets:
# - name: mysa-token-t5tgf

kubectl get secret mysa-token-t5tgf -oyaml
# apiVersion: v1
# data:
#   ca.crt: ..
#   namespace: ...
#   token: ...
# kind: Secret
# metadata:
#   annotations:
#     kubernetes.io/service-account.name: mysa
#     kubernetes.io/service-account.uid: 26a257cc-8921-4e38-81a0-be5b2f47abe5
#   creationTimestamp: "2021-06-12T11:56:24Z"
#   name: mysa-token-t5tgf
#   namespace: default
#   resourceVersion: "365"
#   selfLink: /api/v1/namespaces/default/secrets/mysa-token-t5tgf
#   uid: 6b9f10bf-42cb-400a-b97d-0d64e1274054
# type: kubernetes.io/service-account-token

TOKEN=$(kubectl get secret $(kubectl get sa mysa -ojsonpath="{.secrets[0].name}") -ojsonpath="{.data.token}" | base64 -d)
echo $TOKEN
```

```bash
# 헤더 없이 접속
curl -kv https://127.0.0.1:6443/api
# HTTP/1.1 401 Unauthorized
# Www-Authenticate: Basic realm="kubernetes-master"
# {
#   "kind": "Status",
#   "apiVersion": "v1",
#   "metadata": {
#   },
#   "status": "Failure",
#   "message": "Unauthorized",
#   "reason": "Unauthorized",
#   "code": 401
# }

# basic auth 설정
curl -kv -H "Authorization: Bearer $TOKEN" https://127.0.0.1:6443/api
# HTTP/1.1 200 OK
# {
#   "kind": "APIVersions",
#   "versions": [
#     "v1"
#   ],
#   "serverAddressByClientCIDRs": [
#     {
#       "clientCIDR": "0.0.0.0/0",
#       "serverAddress": "172.31.17.32:6443"
#     }
#   ]
# }
```

```bash
kubectl config set-credentials bearer --token=$TOKEN
kubectl config set-context kubernetes-admin@cluster.local --user=bearer


kubectl get pod
# Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:mysa" cannot list resource "pods" in API group "" in the namespace "default"

# 원복
kubectl config set-context kubernetes-admin@cluster.local --user=kubernetes-admin
```


### X.509 인증서

```bash
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssljson

chmod +x cfssl cfssljson
sudo mv cfssl cfssljson /usr/local/bin/
```

```bash
# 사용자 CSR 파일 생성
cat > client-cert-csr.json <<EOF
{
  "CN": "client-cert",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:masters"
    }
  ]
}
EOF
```


```bash
cat > rootCA-config.json <<EOF
{
  "signing": {
    "profiles": {
      "root-ca": {
        "usages": ["signing", "key encipherment", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

```bash
sudo cfssl gencert \
  -ca=/etc/kubernetes/ssl/ca.crt \
  -ca-key=/etc/kubernetes/ssl/ca.key \
  -config=rootCA-config.json \
  -profile=root-ca \
  client-cert-csr.json | cfssljson -bare client-cert

ls -al
# client-cert-csr.json
# client-cert-key.pem
# client-cert.csr
# client-cert.pem
# rootCA-config.json
```

```bash
kubectl config set-credentials x509 \
          --client-certificate=client-cert.pem \
          --client-key=client-cert-key.pem \
          --embed-certs=true
# User "x509" set.

kubectl config set-context kubernetes-admin@cluster.local --user=x509
# Context "default" modified.

# client-certificate과 key가 base64로 인코딩되어 삽입되어 있습니다.
cat $HOME/.kube/config
# apiVersion: v1
# clusters:
# - cluster:
#     certificate-authority-data: LS0tLS1CRUdJTiB...g==
#     server: https://127.0.0.1:6443
#   name: default
# contexts:
# - context:
#     cluster: default
#     user: x509
#   name: default
# current-context: default
# kind: Config
# preferences: {}
# users:
# - name: x509
#   user:
#     client-certificate-data: LS0tLS1CRUdJTiB...
#     client-key-data: LS0tLS1CRUdJTiBSU0EgUFJ...
```


```bash
# 생성
kubectl run nginx --image nginx
# pod/nginx created

# 조회
kubectl get pod
# NAME     READY   STATUS    RESTARTS   AGE
# nginx    1/1     Running   0          5s

# 삭제
kubectl delete pod nginx
# pod/nginx deleted
```

## 역할 기반 접근 제어 (RBAC)

### Role (ClusterRole)

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-viewer
  namespace: default
rules:
- apiGroups: [""] # ""은 core API group을 나타냅니다.
  resources: 
  - pods
  verbs: 
  - get
  - watch
  - list
```

```bash
kubectl apply -f role.yaml
# role.rbac.authorization.k8s.io/pod-viewer created

kubectl get role
# NAME         CREATED AT
# pod-viewer   2020-05-31T17:16:47Z
```

### Subjects

```bash
kubectl get serviceaccount  # 또는 sa
# NAME      SECRETS   AGE
# default   1         28h
# mysa      1         28h

kubectl get serviceaccount mysa -oyaml
# apiVersion: v1
# kind: ServiceAccount
# metadata:
#   creationTimestamp: "2020-06-07T10:08:29Z"
#   name: mysa
#   namespace: default
#   resourceVersion: "292"
#   selfLink: /api/v1/namespaces/default/serviceaccounts/mysa
#   uid: 0183509b-2e36-412d-b229-048f09b2afc1
# secrets:
# - name: mysa-token-vkrsk
```

### RoleBinding (ClusterRoleBinding)

```yaml
# read-pods.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: mysa
roleRef:
  kind: Role
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f read-pods.yaml
# rolebinding.rbac.authorization.k8s.io/read-pods created

kubectl get rolebinding
# NAME        ROLE              AGE
# read-pods   Role/pod-viewer   20s


kubectl config set-context kubernetes-admin@cluster.local --user=bearer

kubectl get pod

# 원복
kubectl config set-context kubernetes-admin@cluster.local --user=kubernetes-admin
```

```yaml
# nginx-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-sa
spec:
  containers:
  - image: nginx
    name: nginx
  serviceAccountName: mysa
```

```bash
kubectl apply -f nginx-sa.yaml
# pod/nginx-sa created

kubectl get pod nginx-sa -oyaml | grep serviceAccountName
#   serviceAccountName: mysa

# 내부 접근
kubectl exec -it nginx-sa -- bash
```

```bash
# kubectl 설치
$ root@nginx-sa:/# curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl \
                 && chmod +x ./kubectl \
                 && mv ./kubectl /usr/local/bin

# Pod 리소스 조회
$root@nginx-sa:/# kubectl get pod
# NAME      READY   STATUS    RESTARTS   AGE
# nginx-sa  2/2     Running   0          2d21h

# Service 리소스 조회
$root@nginx-sa:/# kubectl get svc
# Error from server (Forbidden): services is forbidden: User 
# "system:serviceaccount:default:mysa" cannot list resource "services"
#  in API group "" in the namespace "default"
```

```bash
# Pod를 빠져나갑니다.
$root@nginx-sa:/# exit

# 호스트 서버에서 다음 명령을 수행
cat << EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-viewer
rules:
- apiGroups: [""]
  resources: 
  - pods
  - services   # services 리소스 추가
  verbs: 
  - get
  - watch
  - list
EOF
# role.rbac.authorization.k8s.io/pod-viewer edited

# 다시 Pod 접속
kubectl exec -it nginx-sa -- bash

# Service 리소스 조회
$root@nginx-sa:/# kubectl get svc
# NAME           TYPE           CLUSTER-IP      ...   
# kubernetes     ClusterIP      10.43.0.1       ...

$root@nginx-sa:/# exit
```

### Clean up

```bash
kubectl delete pod nginx-sa
kubectl delete rolebinding read-pods
kubectl delete role pod-viewer
kubectl delete sa mysa
```

## 네트워크 접근 제어 (Network Policy)

### 쿠버네티스 네트워크 기본 정책

### `NetworkPolicy` 문법

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: 
  - {}
  egress: 
  - {}
```

### 네트워크 구성

```bash
kubectl run client --image nginx
# pod/client created 
```

#### Inbound / Outbound

```yaml
# in-out.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: in-out
  namespace: default
spec:
  podSelector: {}
  ingress: []
  egress:
  - {}
```

```bash
kubectl apply -f in-out.yaml
# networkpolicy.networking.k8s.io/in-out created

kubectl get networkpolicy
# NAME       POD-SELECTOR   AGE
# in-out     <none>         2s

kubectl get networkpolicy in-out -oyaml
# apiVersion: networking.k8s.io/v1
# kind: NetworkPolicy
# metadata:
#   ...
# spec:
#   podSelector: {}
#   policyTypes:
#   - Ingress
```

#### Web pod 오픈

```yaml
# web-open.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-open
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: web
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 80
```

```bash
kubectl apply -f web-open.yaml
# networkpolicy.networking.k8s.io/deny-all created

# run=web 이라는 라벨을 가진 웹 서버 생성
kubectl run web --image nginx
# pod/web created 

# run=non-web 이라는 라벨을 가진 웹 서버 생성
kubectl run non-web --image nginx
# pod/non-web created 

# Pod IP 확인
kubectl get pod -owide
# NAME      READY   STATUS    RESTARTS   AGE     IP            NODE 
# web       1/1     Running   0          34s     10.42.0.169   master
# non-web   1/1     Running   0          32s     10.42.0.170   master
# client    1/1     Running   0          28s     10.42.0.171   master
```

```bash
# client Pod 진입
kubectl exec -it client -- bash

# web Pod 호출
$root@client:/# curl 10.42.0.169  # web
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...

# non-web Pod 호출
$root@client:/# curl 10.42.0.170  # non-web
# curl: (7) Failed to connect to 10.42.0.170 port 80: Connection refused

$root@client:/# exit
```

#### Web과의 통신만 허용된 app

```yaml
# allow-from-web.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-from-web
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: app
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: web
```

```bash
kubectl apply -f allow-from-web.yaml
# networkpolicy.networking.k8s.io/allow-from-web created

# run=app 이라는 라벨을 가진 앱 서버 생성
kubectl run app --image nginx
# pod/app created

# Pod IP 확인
kubectl get pod -owide
# NAME      READY   STATUS    RESTARTS   AGE     IP            NODE
# web       1/1     Running   0          34s     10.42.0.169   master
# non-web   1/1     Running   0          32s     10.42.0.170   master
# client    1/1     Running   0          28s     10.42.0.171   worker
# app       1/1     Running   0          28s     10.42.0.172   worker

# client Pod 진입
kubectl exec -it client -- bash

# client에서 app 서버 호출
$root@client:/# curl 10.42.0.172    # app
# curl: (7) Failed to connect to 10.42.0.172 port 80: Connection refused

# client Pod 종료
$root@client:/# exit

# web Pod 진입
kubectl exec -it web -- bash

# web에서 app 서버 호출
$root@web:/# curl 10.42.0.172
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...

# web Pod 종료
$root@web:/# exit

# non-web Pod 진입
kubectl exec -it non-web -- bash

# non-web에서 app 서버 호출
$root@non-web:/# curl 10.42.0.172
# curl: (7) Failed to connect to 10.42.0.172 port 80: Connection refused

$root@non-web:/# exit
```

#### dev 네임스페이스 외의 아웃바운드 차단

```yaml
# dont-leave-dev.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: dont-leave-dev
  namespace: dev
spec:
  podSelector: {}
  ingress:
  - {}
  egress:
  - to:
    - podSelector: {}
```

```bash
# dev 네임스페이스 생성
kubectl create ns dev
# namespace/dev created

# Egress 네트워크 정책 생성
kubectl apply -f dont-leave-dev.yaml
# networkpolicy.networking.k8s.io/dont-leave-dev created

kubectl run dev1 --image nginx -n dev
# pod/dev1 created

kubectl run dev2 --image nginx -n dev
# pod/dev2 created

# Pod IP 확인
kubectl get pod -owide -n dev
# NAME      READY   STATUS    RESTARTS   AGE     IP            NODE 
# dev1      1/1     Running   0          34s     10.42.0.191   master 
# dev2      1/1     Running   0          32s     10.42.0.192   master 

# dev1 Pod 진입
kubectl exec -it dev1 -n dev -- bash

# dev1에서 dev2로 호출
$root@dev1:/# curl 10.42.0.192
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...

# dev1에서 proxy 서버로 호출 (proxy 서버IP: 10.42.0.183)
$root@dev1:/# curl 10.42.0.183
# curl: (7) Failed to connect to 10.42.0.183 port 80: Connection refused

$root@dev1:/# exit
```

### 네트워크 정책 전체 스펙

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: full-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: dev
    - podSelector:
        matchLabels:
          role: web
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 53
```

### Clean up

```bash
kubectl delete pod app client non-web web
kubectl delete networkpolicy --all -A
kubectl delete ns dev
```
