# k8s 관리 도구

## k8s Dashboard

* k8s 클러스터를 위한 범용 웹 기반 UI 제공
* k8s 환경관리, 문제 해결, Monitoring 하는 직관적인 접근 방식 제공 
* 모든 노드에서 메모리 및 CPU 사용량을 포함한 기본 지표에 접근 가능

k8s Dashboard 설치

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml 
```

API Server 를 통한 보안 접속

```
# cluster role 확인
$ kubectl get clusterrole cluster-admin
NAME            CREATED AT
cluster-admin   2026-02-05T05:48:23Z

$ kubectl describe clusterrole cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
```
             
```
# dashboard 를 위한 어드민 유저 생성
$ kubectl apply -f dashboard_rbac/dashboard-admin-user.yaml

# cluster-admin 의 권한을 dashboard 어드민 유저 권한에 바인딩
$ kubectl apply -f dashboard_rbac/clusterrolebinding-admin-user.yaml

$ kubectl -n kubernetes-dashboard get sa
NAME                   SECRETS   AGE
dashboard-admin-user   0         2m7s
default                0         13m
kubernetes-dashboard   0         13m

# 어드민 유저로 접근을 위한 token 생성
$ kubectl -n kubernetes-dashboard create token dashboard-admin-user
```

```
# 보안 접속을 위한 인증서 생성 

$ grep 'client-certificate-data' ~/.kube/config | head -n 1 \
| awk '{print $2}' | base64 -d > kubecfg.crt

$ grep 'client-key-data' ~/.kube/config | head -n 1 \
| awk '{print $2}' | base64 -d > kubecfg.key

$ openssl pkcs12 -export -clcerts \
  -inkey kubecfg.key \
  -in kubecfg.crt \
  -out kubecfg.p12 \
  -name "kubernetes-admin" \
  -passout pass:1234

$ sudo cp /etc/kubernetes/pki/ca.crt .
```

```
# 생성한 인증서들 (kubecfg.crt, kubecfg.key, kubecfg.p12, ca.crt) 접속할 PC 로 이동하여 등록
# 접속 PC 에서 실행 (Mac 기준)

# 루트 CA 신뢰
$ sudo security add-trusted-cert \
  -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  ca.crt
$ security find-certificate -c "CA"

# 개인 인증서 / 키 등록 (Mac 기준)
$ security import kubecfg.crt \
  -k ~/Library/Keychains/login.keychain-db
$ security import kubecfg.key \
  -k ~/Library/Keychains/login.keychain-db \
  -T /Applications/Safari.app \
  -T /Applications/Google\ Chrome.app

```

```
# dashboard 접속

https://192.168.56.100:6443/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
```

![img.png](img1_1.png)
![img_1.png](img1_2.png)

