# 12장 워크플로우 관리

### Argo workflow 설치

- https://argoproj.github.io/argo-workflows/

```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-workflows/stable/manifests/quick-start-postgres.yaml

kubectl create rolebinding default-admin --clusterrole=admin \
      --serviceaccount=default:default
```

```yaml
# argo-ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: http-issuer
    kubernetes.io/ingress.class: nginx
    ingress.kubernetes.io/protocol: https
    nginx.ingress.kubernetes.io/backend-protocol: https
  name: argo-server
spec:
  rules:
  - host: argo.10.0.1.1.sslip.io
    http:
      paths:
      - backend:
          serviceName: argo-server
          servicePort: 2746
        path: /
  tls:
  - hosts:
    - argo.10.0.1.1.sslip.io
    secretName: argo-tls
```

```bash
kubectl apply -f argo-ingress.yaml
```

### Argo workflow

```yaml
# single-job.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-
  namespace: default
spec:
  entrypoint: whalesay
  templates:
  - name: whalesay
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["hello world"]
```

```bash
# Workflow 생성
kubectl create -f single-job.yaml
# workflow.argoproj.io/hello-world-tcnjj created

kubectl get wf
# NAME                STATUS      AGE
# hello-world-tcnjj   Succeeded   17s

kubectl get pod
# NAME                READY   STATUS      RESTARTS   AGE
# hello-world-tcnjj   0/2     Completed   0          31s

kubectl logs hello-world-tcnjj -c main
#  _____________
# < hello world >
#  -------------
#     \
#      \
#       \
#                     ##        .
#               ## ## ##       ==
#            ## ## ## ##      ===
#        /""""""""""""""""___/ ===
#   ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
#        \______ o          __/
#         \    \        __/
#           \____\______/
# 
# 
# Hello from Docker!
# This message shows that your installation appears to be working 
```

### 파라미터 전달

```yaml
# param.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-parameters-
  namespace: default
spec:
  entrypoint: whalesay
  arguments:
    parameters:
    - name: message
      value: hello world through param

  templates:
  ###############
  # entrypoint
  ###############
  - name: whalesay
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]
```

```bash
kubectl create -f param.yaml
```

### Sequential step 실행

```yaml
# sequential.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: serial-step-
  namespace: default
spec:
  entrypoint: hello-step
  templates:

  ###############
  # template job
  ###############
  - name: whalesay
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]

  ###############
  # entrypoint
  ###############
  - name: hello-step
    # 순차 실행
    steps:
    - - name: hello1
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello1"
    - - name: hello2
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello2"
    - - name: hello3
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello3"
```

```bash
kubectl create -f sequential.yaml
```

### Parallel step 실행

```yaml
# parallel.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: parallel-steps-
  namespace: default
spec:
  entrypoint: hello-step
  templates:

  ###############
  # template job
  ###############
  - name: whalesay
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]

  ###############
  # entrypoint
  ###############
  - name: hello-step
    # 병렬 실행
    steps:
    - - name: hello1
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello1"
    - - name: hello2
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello2"
      - name: hello3        # 기존 double dash에서 single dash로 변경
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello3"
```

- double-dash: 앞의 job을 이어서 순차적으로 실행(직렬)
- single-dash: 앞의 job과 동시에 병렬적으로 실행(병렬)

```bash
kubectl create -f parallel.yaml
```

### 복잡한 DAG 실행

```yaml
# dag.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-diamond-
  namespace: default
spec:
  entrypoint: diamond
  templates:
  
  ###############
  # template job
  ###############
  - name: echo
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:3.7
      command: [echo, "{{inputs.parameters.message}}"]

  ###############
  # entrypoint
  ###############
  - name: diamond
    # DAG 구성
    dag:
      tasks:
      - name: A
        template: echo
        arguments:
          parameters: [{name: message, value: A}]
      - name: B
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: B}]
      - name: C
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: C}]
      - name: D
        dependencies: [B, C]
        template: echo
        arguments:
          parameters: [{name: message, value: D}]
```

```bash
kubectl create -f dag.yaml
```

### 다양한 Examples

- https://argoproj.github.io/argo-workflows/examples/

### Clean up

```bash
kubectl delete wf --all
kubectl delete -f https://raw.githubusercontent.com/argoproj/argo-workflows/stable/manifests/quick-start-postgres.yaml
```
