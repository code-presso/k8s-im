# 03장 Helm 패키지매니저

### helm 설치

```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

### helm chart 생성

```bash
# Creating mychart
helm create mychart

ls mychart
# Chart.yaml  charts  templates  values.yaml
```

```bash
ls mychart/templates
# NOTES.txt  _helpers.tpl  deployment.yaml  hpa.yaml  ingress.yaml  service.yaml  serviceaccount.yaml  tests
```


```yaml
# mychart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}         # 서비스 타입 지정
  ports:
    - port: {{ .Values.service.port }}     # 서비스 포트 지정
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "mychart.selectorLabels" . | nindent 4 }}
```

### helm chart 

```bash
helm install foo ./mychart
```

```bash
helm list

helm list -n kube-system
```



```bash
helm template foo ./mychart > foo-output.yaml

cat foo-output.yaml
```

```bash
kubectl get svc
```

```yaml
# mychart/values.yaml
...

service:
  type: NodePort    # 기존 ClusterIP
  port: 80        
...
```


```bash
helm upgrade foo ./mychart

kubectl get svc

helm list
```


```bash
helm status foo
```


```bash
helm delete foo

helm list
```

### helm 외부 저장소

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
```
#### helm 외부 저장소 검색

```bash
helm search repo bitnami

helm search repo bitnami/airflow
```

#### helm 외부 저장소 

```bash
helm install mynginx bitnami/nginx
helm delete mynginx
```

#### helm 외부 chart download

```bash
helm fetch --untar bitnami/nginx

ls nginx/
vim nginx/values.yaml
# serviceType: LoadBalancer --> NodePort
helm install mynginx ./nginx

kubectl get svc
```

외부 저장소 허브: https://artifacthub.io

### Clean up

```bash
helm delete mynginx
```
