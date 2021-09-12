# 09장 사용자 정의 리소스

```yaml
# mypod-crd.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: mypods.crd.example.com
spec:
  group: crd.example.com
  version: v1
  scope: Namespaced
  names:
    plural: mypods   # 복수 이름
    singular: mypod  # 단수 이름
    kind: MyPod      # Kind 이름
    shortNames:      # 축약 이름
    - mp
```

```bash
# crd 생성
kubectl apply -f mypod-crd.yaml
# customresourcedefinition.apiextensions.k8s.io/mypods.crd.example.com created

# crd 리소스 확인
kubectl get crd | grep mypods
# NAME                     CREATED AT          
# mypods.crd.example.com   2020-06-14T09:33:32Z          
```

```bash
# MyPod 리소스 생성
cat << EOF | kubectl apply -f -
apiVersion: "crd.example.com/v1"
kind: MyPod
metadata:
  name: mypod-test
spec:
  uri: "any uri"
  customCommand: "custom command"
  image: nginx
EOF
# mypod.crd.example.com/mypod-test created

kubectl get mypod
# NAME         AGE
# mypod-test   3s

# 축약형인, mp로도 조회가 가능합니다.
kubectl get mp
# NAME         AGE
# mypod-test   3s

# MyPod의 상세 정보를 조회합니다.
kubectl get mypod mypod-test -oyaml
# apiVersion: crd.example.com/v1
# kind: MyPod
# metadata:
#   ...
#   name: mypod-test
#   namespace: default
#   resourceVersion: "723476"
#   selfLink: /apis/crd.example.com/v1/namespaces/default/mypods/mypod-test
#   uid: 50dd0cc8-0c1a-4f43-854b-a9c212e2046d
# spec:
#   customCommand: custom command
#   image: nginx
#   uri: any uri

# MyPod를 삭제합니다.
kubectl delete mp mypod-test
# mypod.crd.example.com "mypod-test" deleted
```

### Custom Controller

```bash
# MyPod 정의
struct MyPod {
  Uri string
  CustomCommand string
  ...
}


def main {
  # 무한루프
    for {
      # 신규 이벤트
        desired := apiServer.getDesiredState(MyPod)
        # 기존 이벤트
        current := apiServer.getCurrentState(MyPod)
        
        # 변경점 발생시(수정,생성,삭제), 특정 동작 수행
        if desired != current {
            makeChanges(desired, current) 
        }
    }
}
```
