# Pod 네트워킹을 위한 Service 생성과 관리

## Service object가 필요한 이유

### k8s Service

* Service는 네트워크 추상화 object로, 생성된 Pod에 동적 접근이 가능하고, 이를 통해 애플리케이션을 클러스터 내의 네트워크 서비스로 노출할 수 있다.
* Service는 IP 주소 또는 DNS 이름을 통해 특정 포트에 직접 액세스할 수 있도록 Pod를 논리적으로 그룹화할 수 있다.
* k8s는 Pod에게 고유한 IP 주소를 할당하고, Service를 생성하여 Pod 집합에 대한 단일 DNS 명을 부여하여 부하 분산을 수행할 수 있다.


### Service는 왜 필요할까

![img.png](img.png)

* Pod에 문제가 생기면 k8s 는 그 Pod를 삭제 후 재생성한다.
* 이 때 기존 Pod IP가 같을 확률은 매우 낮다.
* k8s 설계 사상은 Pod는 언제든 장애에 의해서 Down 될 수 있고, 신규 Pod는 새로운 IP가 부여되기 때문에 애플리케이션을 매번 수정을 해야 하는 불편함이 있는데, 
이를 없애기 위해 Service를 연결하여 사용하게 되었다.
* 클러스터의 외부에서 내부의 Pod 애플리케이션 접근을 위한 단일 진입점 역할을 한다.
* Proxy(Load Balancer)의 역할처럼 연결된 Pod 들에 트래픽을 전달하는 object 가 필요
* Service object는 연결된 각각의 Pod로 Client 요청 트래픽을 포워딩 해주는 Proxy와 같다.


### Service는 어떻게 연결되나?

![img_1.png](img_1.png)

* Service는 가상 IP 와 port를 갖고 생성
* kube-proxy는 이 Service의 가상 IP를 구현하고 port와 함께 관리하는 역할
* Service object는 Pod를 트래픽에 노출, Endpoint object에는 Pod IP 주소와 port 포함
* Service를 사용하여 Pod를 노출하면 kube-proxy는 Service object와 그룹화된 Pod로 트래픽을 보내는 네트워크 규칙(Rules) 생성
* kube-proxy는 Service 변경 사항 및 Endpoint 모니터링 후 설정된 mode(iptables, IPVS)를 사용하여 Service에 연결된 Pod로 트래픽을 routing 하기 위한 rule 생성, 업데이트

![img_2.png](img_2.png)


## Service type

* ClusterIP : Service를 클러스터 내부 가상 IP를 구현하여 노출
* NodePort : 고정 포트(30000 ~ 32767)로 각 Node의 IP에 Service를 외부에 노출
* LoadBalancer : 클라우드 공급자(CSP)의 로드 밸런서를 사용하여 Service를 외부에 노출
* ExternalName : 일반적인 selector 연결이 아닌 외부 서비스에 대한 DNS name을 제공해 내부 파드가 외부의 특정 도메인에 접근하기 위한 리소스


* Ingress : Ingress를 사용하여 Service를 외부에 노출, Ingress는 서비스 유형은 아니지만, 클러스터의 진입점 역할을 수행


### ClusterIP

* ClusterIP 서비스는 외부에 노출될 필요가 없는 애플리케이션의 내부 통신을 위한 타입
* 예를들어, backend 서비스에서 DB 서비스로 데이터 요청 처리를 하는 내부 연결 애플리케이션인 경우 ClusterIP 서비스를 사용하여 연결
* 서비스 type 의 default type

```
apiVersion: v1
kind: Service
metadata:
  name: clusterip-svc
spec:
  selector:
    app: backend
  ports:
    - protocol : TCP
      port: 9000
      targetPort: 8000
  type: ClusterIP
```

### NodePort

![img_3.png](img_3.png)

* NodePort는 애플리케이션에 대한 외부 연결을 활성화하여 ClusterIP 서비스의 기능을 확장
* NodePort 서비스를 생성하면 클러스터의 모든 Node에 특정한 포트(30000 ~ 32767)를 열어 외부에서 접근, 생성된 Service의 ClusterIP로 트래픽 routing
* 이 때 kube-proxy(Service -> Pod)가 이를 forwarding 하도록 netfilter의 chain rule 수정
* 웹 애플리케이션이나 API와 같이 클러스터 외부에서 액세스해야 하는 애플리케이션에 적합
* 외부에서 <NodeIP>:<NodePort>로도 접근 가능
* Cluster 내부에서 <내부IP>:<port> 접속 가능

```
apiVersion: v1
kind: Service
metadata:
  name: nodeport-svc
spec:
  selector:
    app: backend
  ports:
    - protocol : TCP
      port: 9000
      targetPort: 8000
      nodePort: 30001 # 지정안하면 랜덤
  type: NodePort
```

externalTrafficPolicy : { Local | Cluster }
* Local 정책으로 변경하면, 외부 요청에 대해 Cluster 정책처럼 무작위 트래픽 분산이 되지 않고, 처음 들어온 node의 pod로만 지속적으로 연결됨
  (kube-proxy는 proxy 요청을 Local endpoint 로만 proxy하고 트래픽을 다른 node로 전달 안함)


internalTrafficPolicy : { Local | Cluster }
* 내부 요청에 대해 Service에 그룹화된 Pod 중 동일 Node에 존재하는 Pod 간의 트래픽 정책


sessionAffinity : ClientIP 로 주면, 같은 클라이언트 IP는 같은 Pod로 감

```
...
spec:
  clusterIP: 10.96.100.100
  clusterIPs:
  - 10.96.100.100
  externalTrafficPolicy: cluster
  internalTrafficPolicy: cluster
...
```


## LoadBalancer




