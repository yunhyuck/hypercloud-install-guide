# HyperAuth 설치 가이드

## 구성 요소 및 버전
* hyperauth
    * [tmaxcloudck/hyperauth:b1.0.3.4](https://hub.docker.com/layers/tmaxcloudck/hyperauth/b1.0.3.4/images/sha256-658f98c01c29b30271596c4f61d072c61778cb3e9ae58ffdc96a56b4fdbad4f7?context=explore)

## Prerequisites
X

## Dependencies
openssl binary

csi-cephfs-sc storageClass

LoadBalancer type의 service 생성 가능

## 폐쇄망 설치 가이드



## Install Steps
1. [초기화 작업](https://github.com/tmax-cloud/hypercloud-install-guide/blob/4.1/HyperAuth/README.md#step-1-%EC%B4%88%EA%B8%B0%ED%99%94-%EC%9E%91%EC%97%85)
2. [SSL 인증서 생성](https://github.com/tmax-cloud/hypercloud-install-guide/blob/4.1/HyperAuth/README.md#step-2-ssl-%EC%9D%B8%EC%A6%9D%EC%84%9C-%EC%83%9D%EC%84%B1)
3. [HyperAuth Deployment 생성](https://github.com/tmax-cloud/hypercloud-install-guide/blob/4.1/HyperAuth/README.md#step-3-hyperauth-deployment-%EB%B0%B0%ED%8F%AC)
4. [Kubernetes OIDC 연동]()

## Step 1. 초기화 작업
* 목적 : `HyperAuth 구축을 위한 초기화 작업 및 DB 구축`
* 생성 순서 : [1.initialization.yaml](manifest/1.initialization.yaml) 실행 `ex) kubectl apply -f 1.initialization.yaml`)
* 비고 : 아래 명령어 수행 후, Postgre DB table 생성 확인 (약 96-97개)
```bash
    $ kubectl exec -it $(kubectl get pods -n hyperauth | grep postgre | cut -d ' ' -f1) -n hyperauth -- bash
    $ psql -U keycloak keycloak
    $ \dt
 ```

## Step 2. SSL 인증서 생성
* 목적 : `HTTPS 인증을 위한 openssl 인증서를 생성하고 secret으로 변환`
* 생성 순서 : 아래 명령어를 실행하여 인증서 생성 및 secret을 생성 (Master Node의 특정 directory 내부에서 실행 권장)
```bash
    $ openssl req -newkey rsa:4096 -nodes -sha256 -keyout hyperauth.key -x509 -subj "/C=KR/ST=Seoul/O=tmax/CN=$(kubectl describe service hyperauth -n hyperauth | grep 'LoadBalancer Ingress' | cut -d ' ' -f7)" -days 365 -config <(cat /etc/ssl/openssl.cnf <(printf "[v3_ca]\nsubjectAltName=IP:$(kubectl describe service hyperauth -n hyperauth | grep 'LoadBalancer Ingress' | cut -d ' ' -f7)")) -out hyperauth.crt
    $ kubectl create secret tls hyperauth-https-secret --cert=./hyperauth.crt --key=./hyperauth.key -n hyperauth
    $ cp hyperauth.crt /etc/kubernetes/pki/hyperauth.crt
```
* 비고 : 
    * Kubernetes Master가 다중화 된 경우, hyperauth.crt를 각 Master 노드들의 /etc/kubernetes/pki/hyperauth.crt 로 cp


## Step 3. HyperAuth Deployment 배포
* 목적 : `HyperAuth 설치`
* 생성 순서 :
    * [2.hyperauth_deployment.yaml](manifest/2.hyperauth_deployment.yaml) 실행 `ex) kubectl apply -f 2.hyperauth_deployment.yaml`
    * HyperAuth Admin Console에 접속
        * `kubectl get svc hyperauth -n hyperauth` 명령어로 IP 확인
        * 계정 : admin/admin
    * Manage > Users > Add user 에서 admin-tmax.co.kr 계정을 생성
        * Name : admin-tmax.co.kr
        * Email : 관리자 전용 email
    * Manage > Users > admin-tmax.co.kr UserDetail > Credentials 에서 password 재설정
    * Master > Add realm > Import - Select file 에 [3.tmax_realm_export.json](manifest/3.tmax_realm_export.json) 을 추가하여 Realm Import


## Step 4. Kubernetes OIDC 연동
* 목적 : `Kubernetes의 RBAC 시스템과 HyperAuth 인증 연동`
* 생성 순서 :
    * Kubernetes Cluster Master Node에 접속
    * {HYPERAUTH_SERVICE_IP} = $(kubectl describe service hyperauth -n hyperauth | grep 'LoadBalancer Ingress' | cut -d ' ' -f7)
    * `/etc/kubernetes/manifests/kube-apiserver.yaml` 의 spec.containers[0].command[] 절에 아래 command를 추가
    
    ```yaml
    --oidc-issuer-url=https://{HYPERAUTH_SERVICE_IP}/auth/realms/tmax
    --oidc-client-id=hypercloud4
    --oidc-username-claim=preferred_username
    --oidc-username-prefix=-
    --oidc-groups-claim=group
    --oidc-ca-file=/etc/kubernetes/pki/hyperauth.crt
    ```
    
* 비고 :
    * 자동으로 kube-apiserver 가 재기동 됨