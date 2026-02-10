## Label 활용

* 복잡하고, 다양한 Pod를 호율적인 집합으로 다루기 위한 방법으로 Label 사용
* Label은 key-value 기반의 속성 tag로 하나 이상 설정 가능
* 용도에 따른 리소스 선택 시 유용 -> 객체를 식별하고 그룹화
* Pod는 Label을 가질 수 있고, Label 검색 조건에 따라서 특정 Label을 가지고 있는 Pod만을 선택할 수 있다.
* Label을 선택하여 특정 리소스만 배포하거나 업데이트 할 수 있고, 
* Label로 선택된 리소스만 Service에 연결하거나 특정 Label로 선택된 리소스에만 네트워크 접근 권한을 부여하는 등의 작업을 할 수 있다.

```
-- Pod YAML
metadata:
  name: mynode-pod
  labels:
    app: run-node
    job: front
```

Label 예제

1. 3개의 pod에 같은 Label 을 주고 Service 에서 해당 Label를 연결하면, 서비스로 들어오는 요청이 3개의 pod로 분산되어 전달된다. (로드밸런싱)

![img.png](img.png)

```
$ kubectl create namespace infra-team-ns1
$ kubectl apply -f label-pod.yaml

$ kubectl get po,svc -o wide -n infra-team-ns1
NAME              READY   STATUS    RESTARTS   AGE     IP              NODE        NOMINATED NODE   READINESS GATES
pod/label-pod-a   1/1     Running   0          5m13s   10.111.156.75   k8s-node1   <none>           <none>
pod/label-pod-b   1/1     Running   0          5m13s   10.111.156.78   k8s-node1   <none>           <none>
pod/label-pod-c   1/1     Running   0          5m13s   10.111.156.77   k8s-node1   <none>           <none>

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE     SELECTOR
service/infra-svc1   ClusterIP   10.97.70.170   <none>        80/TCP    5m13s   type=infra1

$ kubectl -n infra-team-ns1 describe svc infra-svc1
Name:              infra-svc1
Namespace:         infra-team-ns1
Labels:            <none>
Annotations:       <none>
Selector:          type=infra1
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.97.70.170
IPs:               10.97.70.170
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.111.156.75:80,10.111.156.77:80,10.111.156.78:80
Session Affinity:  None
Events:            <none>
```

## Node Schedule

* k8s 배포가 더 크고 다양해지면 스케줄링(노드 할당) 관리가 더 중요해짐
* kube-scheduler 가 Pod가 할당 될 Node를 결정


* 사용자가 원하는 Node에 스케줄링 가능
* nodeSelector : kube-scheduler에게 지정 Node 배치 요청
* nodeName : 해당 Node의 kubelet 에게 직접 요청
* Affinity : 다양한 조건(친밀도, Affinity)로 Node 배치 요청
* Tolerations : Taint(자물쇠)가 설정된 Node 강제 허용 요청 -> Master Node에도 Pod 생성할 때 사용
* schedulerName : Multi Cluster 환경인 경우

hosname 을 이용하여 node1 에 Pod 생성

```
apiVersion: v1
kind: Pod
...
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-node1
  ...
```

label 을 이용하여 지정된 label을 가진 node에 Pod 생성

```
$ kubectl label nodes k8s-node1 type=infra1

apiVersion: v1
kind: Pod
...
spec:
  nodeSelector:
    type: infra1
  ...
  
# label 제거
$ kubectl label nodes k8s-node1 type-
```

nodeName 을 이용하여 Pod 생성 (kubelet이 바로 생성)

```
apiVersion: v1
kind: Pod
...
spec:
  nodeName: k8s-node1
  ...
```

Node1 에 NoSchedule taint 설정 (설정하면 Pod 생성 안됨)

```
# 명령형
$ kubectl taint nodes k8s-node1 kubernetes.io/k8s-node1:NoSchedule
```

```
# 선언형
apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "type"
    operator: : "Equal"
    value: "infra1"
    effect: "NoSchedule"
```

taint 해제

```
$ kubectl taint nodes k8s-node1 kubernetes.io/k8s-node1:NoSchedule-
```

## Probes 이해와 활용



