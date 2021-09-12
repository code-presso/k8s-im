# 04 Ingress

### Ingress Controller 종류

- NGINX Ingress Controller ([https://kubernetes.github.io/ingress-nginx](https://kubernetes.github.io/ingress-nginx/))
- HAProxy ([https://haproxy-ingress.github.io](https://haproxy-ingress.github.io/))
- AWS ALB Ingress Controller ([https://github.com/kubernetes-sigs/aws-alb-ingress-controller](https://github.com/kubernetes-sigs/aws-alb-ingress-controller))
- Ambassador ([https://www.getambassador.io](https://www.getambassador.io))
- Kong ([https://konghq.com](https://konghq.com/))
- traefik ([https://github.com/containous/traefik](https://github.com/containous/traefik))

### NGINX Ingress Controller 설치

```bash
kubectl create ns ctrl

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm fetch --untar ingress-nginx/ingress-nginx
vim ingress-nginx/values.yaml
# change hostPort to "true"
#  hostPort:
#    enabled: false --> true
# change LoadBalancer to ClusterIP
# type: LoadBalancer --> ClusterIP
helm install ingress-nginx ./ingress-nginx -nctrl
```

```bash
kubectl get svc -n ctrl

kubectl get pod -n ctrl -owide
```

## `Ingress` 기본 사용법

### 도메인 주소 테스트

```bash
nslookup 10.0.1.1.sslip.io
```

```bash
nslookup subdomain.10.0.1.1.sslip.io
```


### 첫 `Ingress` 생성


```bash
kubectl run mynginx --image nginx --expose --port 80

kubectl get pod,svc mynginx
```

```yaml
# mynginx-ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: mynginx
spec:
  rules:
  - host: 10.0.1.1.sslip.io
    http:
      paths:
      - path: /
        backend:
          serviceName: mynginx
          servicePort: 80
```

```bash
NEW_IP=''
sed -i 's/10.0.1.1/'$NEW_IP'/g' mynginx-ingress.yaml

kubectl apply -f mynginx-ingress.yaml

kubectl get ingress

curl 10.0.1.1.sslip.io
```


### 도메인 기반 라우팅

```bash
kubectl run apache --image httpd --expose --port 80

kubectl run nginx --image nginx --expose --port 80
```

```yaml
# domain-based-ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: apache-domain
spec:
  rules:
    # apache 서브 도메인
  - host: apache.10.0.1.1.sslip.io
    http:
      paths:
      - backend:
          serviceName: apache
          servicePort: 80
        path: /
---  
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: nginx-domain
spec:
  rules:
    # nginx 서브 도메인
  - host: nginx.10.0.1.1.sslip.io
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
        path: /
```


```bash
sed -i 's/10.0.1.1/'$NEW_IP'/g' domain-based-ingress.yaml

kubectl apply -f domain-based-ingress.yaml

curl apache.10.0.1.1.sslip.io

curl nginx.10.0.1.1.sslip.io
```


### Path 기반 라우팅

```yaml
# path-based-ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: / 
  name: apache-path
spec:
  rules:
  - host: 10.0.1.1.sslip.io
    http:
      paths:
      - backend:
          serviceName: apache
          servicePort: 80
        path: /apache
---  
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: / 
  name: nginx-path
spec:
  rules:
  - host: 10.0.1.1.sslip.io
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
        path: /nginx
```

- `nginx.ingress.kubernetes.io/rewrite-target: /`: path 재정의 지시자

```bash
sed -i 's/10.0.1.1/'$NEW_IP'/g' path-based-ingress.yaml
kubectl apply -f path-based-ingress.yaml

curl 10.0.1.1.sslip.io/apache

curl 10.0.1.1.sslip.io/nginx
```

## Basic Auth 설정

### Basic Auth 설정

```bash
sudo apt install -y apache2-utils

htpasswd -cb auth foo bar

kubectl create secret generic basic-auth --from-file=auth

kubectl get secret basic-auth -oyaml
```

```yaml
# apache-auth.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - foo'
  name: apache-auth
spec:
  rules:
  - host: apache-auth.10.0.1.1.sslip.io
    http:
      paths:
      - backend:
          serviceName: apache
          servicePort: 80
        path: /
```

```bash
sed -i 's/10.0.1.1/'$NEW_IP'/g' apache-auth.yaml

kubectl apply -f apache-auth.yaml
curl -I apache-auth.10.0.1.1.sslip.io

curl -I -H "Authorization: Basic $(echo -n foo:bar | base64)" apache-auth.10.0.1.1.sslip.io
```


## TLS 설정

### Self-signed 인증서 설정

#### Self-signed 인증서 생성하기

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=apache-tls.$NEW_IP.sslip.io"

ls
```


```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-certs
  namespace: default
data:
  tls.crt: $(cat tls.crt | base64 | tr -d '\n')
  tls.key: $(cat tls.key | base64 | tr -d '\n')
type: kubernetes.io/tls
EOF
```

#### Ingress TLS 설정하기

```yaml
# apache-tls.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: apache-tls
spec:
  tls:
  - hosts:
      - apache-tls.10.0.1.1.sslip.io
    secretName: my-tls-certs
  rules:
  - host: apache-tls.10.0.1.1.sslip.io
    http:
      paths:
      - path: /
        backend:
          serviceName: apache
          servicePort: 80
```

```bash
sed -i 's/10.0.1.1/'$NEW_IP'/g' apache-tls.yaml
kubectl apply -f apache-tls.yaml
# ingress.networking.k8s.io/apache-tls created
```

### cert-manager를 이용한 인증서 발급 자동화

#### cert-manager 설치

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
```

#### Issuer 생성

```yaml
# http-issuer.yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: http-issuer
spec:
  acme:
    email: mymail@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: issuer-key
    solvers:
    - http01:
       ingress:
         class: nginx
```

```bash
# 아래와 같이 특정 URL에 Let's encrypt가 제공한 토큰을 응답하여 도메인 주소 소유권을 증명
http://<YOUR_DOMAIN>/.well-known/acme-challenge/<TOKEN>
```


```bash
kubectl apply -f http-issuer.yaml
# clusterissuer.cert-manager.io/http-issuer created

kubectl get clusterissuer
# NAME          READY   AGE
# http-issuer   True    2m
```

#### cert-manager가 관리하는 TLS `Ingress` 생성

```yaml
# apache-tls-issuer.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    # 앞서 생성한 발급자 지정
    cert-manager.io/cluster-issuer: http-issuer
  name: apache-tls-issuer
spec:
  rules:
  # 10.0.1.1을 공인IP로 변경해 주세요.
  - host: apache-issuer.10.0.1.1.sslip.io
    http:
      paths:
      - backend:
          serviceName: apache
          servicePort: 80
        path: /
  tls:
  - hosts:
    # 10.0.1.1을 공인IP로 변경해 주세요.
    - apache-issuer.10.0.1.1.sslip.io
    secretName: apache-tls
```

```bash
sed -i 's/10.0.1.1/'$NEW_IP'/g' apache-tls-issuer.yaml

kubectl apply -f apache-tls-issuer.yaml
# ingress.networking.k8s.io/apache-tls-issuer created

watch kubectl get certificate 
```

### Clean up

```bash
kubectl delete ingress --all
kubectl delete pod apache nginx mynginx
kubectl delete svc apache nginx mynginx
```
