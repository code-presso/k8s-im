# 05장 스토리지

## `PersistentVolume`

### hostPath PV

```yaml
# hostpath-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-volume
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp
```

```bash
kubectl apply -f hostpath-pv.yaml
# persistentvolume/my-volume created

kubectl get pv
# NAME        CAPACITY  ACCESS MODES   RECLAIM POLICY     STATUS      CLAIM   STORAGECLASS   REASON   AGE             
# my-volume   1Gi       RWO            Retain             Available           manual                  12s                
```

### NFS PV

```yaml
# nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-nfs
spec:
  storageClassName: nfs
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: <NFS_SERVER_IP>
```

### awsElasticBlockStore PV

```bash
aws ec2 create-volume --availability-zone=eu-east-1a \
  --size=80 --volume-type=gp2
# {
#     "AvailabilityZone": "us-east-1a",
#     "Tags": [],
#     "Encrypted": false,
#     "VolumeType": "gp2",
#     "VolumeId": "vol-1234567890abcdef0",
#     "State": "creating",
#     "Iops": 240,
#     "SnapshotId": "",
#     "CreateTime": "YYYY-MM-DDTHH:MM:SS.000Z",
#     "Size": 80
# }
```

```yaml
# aws-ebs.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: aws-ebs
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```

## PersistentVolumeClaim

```yaml
# my-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f my-pvc.yaml
# persistentvolumeclaim/my-pvc created

# 앞에서 생성한 my-volume을 선점하였습니다.
kubectl get pvc
# NAME     STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-pvc   Bound    my-volume   1Gi        RWO            manual         3s


kubectl get pv
# NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
# my-volume   1Gi        RWO            Retain           Bound    default/my-pvc   manual                  97s
```

```yaml
# use-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: use-pvc
spec:
  containers: 
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: /test-volume
      name: vol
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: my-pvc
```

```bash
kubectl apply -f use-pvc.yaml
# pod/use-pvc created

# 데이터 저장
kubectl exec use-pvc -- sh -c "echo 'hello' > /test-volume/hello.txt"
```

```bash
kubectl delete pod use-pvc
# pod/use-pvc deleted

kubectl apply -f use-pvc.yaml
# pod/use-pvc created

kubectl exec use-pvc -- cat /test-volume/hello.txt
# hello
```

### Clean up

```bash
kubectl delete pod use-pvc
kubectl delete pvc my-pvc
kubectl delete pv my-volume
```

## `StorageClass`

### `StorageClass`

[https://github.com/rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner)

```bash
# install
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

kubectl get sc

kubectl get sc local-path -oyaml
# apiVersion: storage.k8s.io/v1
# kind: StorageClass
# metadata:
#   annotations:
#     objectset.rio.cattle.io/id: ""
#   ...
#   name: local-path
#   resourceVersion: "172"
#   selfLink: /apis/storage.k8s.io/v1/storageclasses/local-path
#   uid: 3aede349-0b94-40c8-b10a-784d38f7c120
# provisioner: rancher.io/local-path
# reclaimPolicy: Delete
# volumeBindingMode: WaitForFirstConsumer
```

```yaml
# my-pvc-sc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc-sc
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f my-pvc-sc.yaml
# persistentvolumeclaim/my-pvc-sc created

kubectl get pvc my-pvc-sc
# NAME         STATUS    VOLUME     CAPACITY   ACCESS MODES    STORAGECLASS   AGE
# my-pvc-sc    Pending                                         local-path     11s
```

```yaml
# use-pvc-sc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: use-pvc-sc
spec:
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: my-pvc-sc
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: vol
```

```bash
# pod 생성
kubectl apply -f use-pvc-sc.yaml
# pod/use-pvc-sc created

# STATUS가 Bound로 변경
kubectl get pvc my-pvc-sc
# NAME         STATUS   VOLUME            CAPACITY    ACCESS MODES   STORAGECLASS   AGE
# my-pvc-sc    Bound    pvc-479cff32-xx   1Gi         RWO            local-path     92s        


kubectl get pv
# NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
# pvc-5beef658-xxxx   1Gi        RWO            Delete           Bound    default/my-pvc-sc   local-path              20s



kubectl get pv pvc-5beef658-xxxx -oyaml
# apiVersion: v1
# kind: PersistentVolume
# metadata:
#     ...
#   name: pvc-b1727544-f4be-4cd6-acb7-29eb8f68e84a
#   ...
# spec:
#   ...
#   hostPath:
#     path: /var/lib/rancher/k3s/storage/pvc-b1727544-f4be-4cd6-acb7-29eb8f68e84a
#     type: DirectoryOrCreate
#   nodeAffinity:
#     required:
#       nodeSelectorTerms:
#       - matchExpressions:
#         - key: kubernetes.io/hostname
#           operator: In
#           values:
#           - worker
#    ...
```

### NFS StorageClass 설정


```bash
git clone https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner.git

cd nfs-ganesha-server-and-external-provisioner

kubectl create -n ctrl -f deploy/kubernetes/deployment.yaml
kubectl create -n ctrl -f deploy/kubernetes/rbac.yaml
kubectl create -n ctrl -f deploy/kubernetes/class.yaml
kubectl patch storageclass example-nfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl create clusterrolebinding nfs-admin --clusterrole cluster-admin --serviceaccount default:nfs-provisioner

# install nfs-util from kubespray
cd ~/kubespray
ansible all -i inventory/mycluster/hosts.yaml -m ansible.builtin.shell --become --become-user=root  -a 'apt update && sudo apt install -y nfs-client'


kubectl get pod -n ctrl 
# NAME                               READY   STATUS    RESTARTS   AGE
# nfs-provisioner-78dc99f5b7-kf8hj   1/1     Running   0          2m15s


# nfs-server-provisioner Service도 있습니다.
kubectl get svc -n ctrl 
# NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
# nfs-provisioner   ClusterIP   10.233.16.80   <none>        2049/TCP,2049/...   3m22s


# 새로운 nfs StorageClass 생성
kubectl get sc
# NAME          PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
# example-nfs   example.com/nfs         Delete          Immediate              false                  3m
# local-path    rancher.io/local-path   Delete          WaitForFirstConsumer   false                  9m11s

```

```yaml
# nfs-sc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-sc
spec:
  storageClassName: example-nfs
  # accessModes를 ReadWriteMany로 변경
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f nfs-sc.yaml
# persistentvolumeclaim/nfs-sc created

# pvc 리소스 확인
kubectl get pvc
# NAME        STATUS   VOLUME             CAPACITY   ACCESS MODES  ...
# my-pvc-sc   Bound    pvc-b1727544-xxx   1Gi        RWO           ...
# nfs-sc      Bound    pvc-49fea9cf-xxx   1Gi        RWO           ...

# pv 리소스 확인
kubectl get pv pvc-49fea9cf-xxx
# NAME                CAPACITY   ACCESS MODES   RECLAIM  POLICY   STATUS    CLAIM            STORAGECLASS   REASON  AGE
# pvc-49fea9cf-xxx    1Gi        RWX            Delete            Bound     default/nfs-sc   nfs                    5m


kubectl get pv pvc-49fea9cf-xxx -oyaml
# apiVersion: v1
# kind: PersistentVolume
# metadata:
#   ...
# spec:
#   accessModes:
#   - ReadWriteMany
#   capacity:
#     storage: 1Gi
#   claimRef:
#     apiVersion: v1
#     kind: PersistentVolumeClaim
#     name: nfs-sc
#     namespace: default
#     resourceVersion: "10084380"
#     uid: 2e95f6c4-2b43-4375-808f-0c93e44a1003
#   mountOptions:
#   - vers=3
#   nfs:
#     path: /export/pvc-2e95f6c4-2b43-4375-808f-0c93e44a1003
#     server: 10.43.248.122
#   persistentVolumeReclaimPolicy: Delete
#   storageClassName: nfs
#   volumeMode: Filesystem
# status:
#   phase: Bound
```

```yaml
# use-nfs-sc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: use-nfs-sc-node1
spec:
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: nfs-sc
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: vol
  nodeSelector:
    kubernetes.io/hostname: node1
---
apiVersion: v1
kind: Pod
metadata:
  name: use-nfs-sc-node2
spec:
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: nfs-sc
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: vol
  nodeSelector:
    kubernetes.io/hostname: node2
```

```bash
kubectl apply -f use-nfs-sc.yaml
# pod/use-nfs-sc-node1 created
# pod/use-nfs-sc-node2 created

kubectl get pod -o wide
# NAME               READY  STATUS    RESTARTS  AGE   IP          NODE
# ...
# use-nfs-sc-node1  1/1    Running   0         19s   10.42.0.8   node1
# use-nfs-sc-node2  1/1    Running   0         19s   10.42.0.52  node2
```

```bash
# master Pod에 index.html 파일을 생성합니다.
kubectl exec use-nfs-sc-node1 -- sh -c \
      "echo 'hello world' >> /usr/share/nginx/html/index.html"

# worker Pod에서 호출을 합니다.
kubectl exec use-nfs-sc-node2 -- curl -s localhost
# hello world
```

### Clean up

```bash
kubectl delete pod use-nfs-sc-node1
kubectl delete pod use-nfs-sc-node2
kubectl delete pod use-pvc-sc
kubectl delete pvc nfs-sc
```

## 쿠버네티스 스토리지 활용

```bash
helm fetch --untar bitnami/minio

vim minio/values.yaml
```

```yaml
...
accessKey: 
  password: "myaccesskey"
secretKey: 
  password: "mysecretkey"
...
persistence:
  storageClass: "example-nfs"
  accessMode: "ReadWriteMany"
  size: 2Gi

ingress:
  # false --> true
  enabled: true
  labels: {}

  # annotation 설정
  annotations:
    kubernetes.io/ingress.class: nginx
  path: /
  hosts:
    - minio.10.0.1.1.sslip.io
```

```bash
NEW_IP=''
sed -i 's/10.0.1.1/'$NEW_IP'/g' minio/values.yaml

helm install minio ./minio

kubectl get pod
# NAME                      READY   STATUS             RESTARTS   AGE
# minio-7f58448457-vctrp    1/1     Running            0          2m

kubectl get pvc
# NAME     STATUS   VOLUME             CAPACITY   ACCESS MODES  STORAGECLASS    AGE
# minio    Bound    pvc-cff81820-xxx   10Gi       RWO           example-nfs     2m40s

kubectl get pv
# NAME               CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS  CLAIM           STORAGECLASS   REASON   AGE 
# pvc-cff81820-xxx   10Gi       RWO            Delete           Bound   default/minio   nfs                     3m
```

### Clean up

```bash
helm delete minio
```
