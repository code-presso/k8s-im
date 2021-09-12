# 10장 로깅 & 모니터링


### 클러스터 레벨 로깅 원리

```bash
docker logs <CONTAINER_ID>
```

```bash
/var/lib/docker/containers/<CONTAINER_ID>/<CONTAINER_ID>-json.log
```

```bash
# nginx라는 컨테이너를 하나 실행하고 CONTAINER_ID 값을 복사합니다.
docker run -d nginx
# 4373b7e095215c23057b1dc4423527239e56a33dbd

# docker 명령을 통한 로그 확인
docker logs 4373b7e095215c23057b1dc4423527239e56a33dbd
# /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will ...
# /docker-entrypoint.sh: Looking for shell scripts in /docker-...
# /docker-entrypoint.sh: Launching /docker-entrypoint.d/...
# ...

# 호스트 서버의 로그 파일 확인
sudo tail /var/lib/docker/containers/4373b7e095215c23057b1dc4423527239e56a33dbd/4373b7e095215c23057b1dc4423527239e56a33dbd-json.log
# {"log":"/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, \
# will attempt to perform configuration\n","stream":"stdout",\
# "time":"2020-07-11T03:22:11.817939191Z"}
# ...

# 컨테이너 정리
docker stop 4373b7e095215c23057b1dc4423527239e56a33dbd
docker rm 4373b7e095215c23057b1dc4423527239e56a33dbd
```

### Install Elastic Stack

[Elastic Cloud K8s](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html)


```bash
kubectl apply -f https://download.elastic.co/downloads/eck/1.6.0/all-in-one.yaml

cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 7.13.1
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
EOF

kubectl get pods
kubectl get service quickstart-es-http

cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
spec:
  version: 7.13.1
  count: 1
  elasticsearchRef:
    name: quickstart
EOF
```

```yaml
# kibana.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: http-issuer
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  name: kibana
spec:
  rules:
  - host: kibana.10.0.1.1.sslip.io
    http:
      paths:
      - backend:
          serviceName: quickstart-kb-http
          servicePort: 5601
        path: /
  tls:
  - hosts:
    - kibana.10.0.1.1.sslip.io
    secretName: kibana-tls
```

```bash
NEW_IP=''
sed -i 's/10.0.1.1/'$NEW_IP'/g' kibana.yaml

kubectl apply -f kibana.yaml

kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
# Enter kibana web with following information:
# - user: elastic
# - pass: above password
```

### Install fluent-bit

```bash
# Password for accessing Elasticsearch
kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}'; echo

helm repo add fluent https://fluent.github.io/helm-charts

helm fetch --untar fluent/fluent-bit
vim fluent-bit/values.yaml
```

```yaml
# values.yaml
  outputs: |
    [OUTPUT]
        Name es
        Match kube.*
        Host quickstart-es-http   # <- Change here
        HTTP_User elastic         # <- Add User elastic
        HTTP_Passwd infL94bxxxx   # <- Add Elasticsearch password
        tls On                    # <- Add this
        tls.verify Off            # <- Add this
        tls.debug 1               # <- Add this
        Logstash_Format On
        Retry_Limit False
```

```bash
# fluent-bit 수정
helm install log ./fluent-bit
```

### Clean up

```bash
kubectl delete ingress kibana
kubectl delete kibana quickstart
kubectl delete elasticsearch quickstart
helm delete log
kubectl delete -f https://download.elastic.co/downloads/eck/1.6.0/all-in-one.yaml
```

## 리소스 모니터링 시스템 구축

### 컨테이너 메트릭 정보 수집 방법

```bash
docker run -d nginx
# 4373b7e095215c23057b1dc4423527239e56a33dbd

docker stats 4373b7e095215c23057b1dc4423527239e56a33dbd
# CONTAINER ID    NAME     CPU %     MEM USAGE / LIMIT     MEM    ...    
# 4af9f73eb06f    dreamy   0.00%     3.227MiB / 7.773GiB   0.04%  ...

docker stop 4373b7e095215c23057b1dc4423527239e56a33dbd
docker rm 4373b7e095215c23057b1dc4423527239e56a33dbd
```

### Prometheus & Grafana 구축

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm fetch --untar prometheus-community/kube-prometheus-stack
```

```yaml
# kube-prometheus-stack/values.yaml
grafana:
  ...
  ingress:
    enabled: true   # 기존 false
    annotations:
      kubernetes.io/ingress.class: nginx   # 추가
    hosts:
    - grafana.10.0.1.1.sslip.io            # 공인IP 입력
```

```bash
helm install prom ./kube-prometheus-stack
# ...

watch kubectl get pod

# Enter grafana.10.0.1.1.sslip.io
- user: admin
- password: prom-operator
```

### Clean up

```bash
helm delete prom
```
