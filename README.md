
## host file 참고

nginx ingress 를 통해 접속할 수 있도록 `로컬 PC` `hosts` 파일 수정  

```
192.168.10.XXX  core.harbor.domain jenkins.cluster.local argocd.example.com minio-example.local console.minio-example.local
```

### 접속방법

<https://core.harbor.domain/>
admin/Harbor12345

<https://jenkins.cluster.local/>
admin/...

<https://argocd.example.com>
admin/...

<https://console.minio-example.local/>
rootuser/rootpass123

<https://kibana-example.es.local/app/home>
elastic/password

## kubadm

> <https://tech.hostway.co.kr/2022/08/30/1374/>

```shell
# 스왑 비활성화 
sudo swapoff -a 

# 의존성 리스트 설지
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add 
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

sudo apt-get install kubeadm kubelet kubectl
sudo apt-mark hold kubeadm kubelet kubectl

# 버전확인
kubeadm version 
# kubeadm version: &version.Info{Major:"1", Minor:"28", GitVersion:"v1.28.2 ...

# Controller Node
sudo kubeadm init --ignore-preflight-errors=all \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.10.XXX \
  --cri-socket /var/run/cri-dockerd.sock

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```shell
# taint all node
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### calico cni install

```shell
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

watch kubectl get pods -n kube-system
NAME                                       READY   STATUS    
calico-kube-controllers-7ddc4f45bc-v5rpc   1/1     Running   
calico-node-mbwnt                          1/1     Running   
coredns-cfbfd9cb6-nhf5s                    1/1     Running   
coredns-cfbfd9cb6-p7h5p                    1/1     Running   
...
```

### ingress controller

> <https://github.com/kubernetes/ingress-nginx>

`random nodePort` 를 고정하기 위해 아래와 같이 수정  

```yaml
apiVersion: v1
kind: Service
metadata:
  ...
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - appProtocol: http
    name: http
    port: 80
    nodePort: 30080 # nodePort 고정
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    port: 443
    nodePort: 30443 # nodePort 고정
    protocol: TCP
    targetPort: https
  ...
```

```shell
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml -o ingress-deploy.yaml
kubectl apply -f ingress-deploy.yaml
```


### rancher storageClass

> <https://github.com/rancher/local-path-provisioner>  

```shell
# stable 버전으로 설치
curl https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.24/deploy/local-path-storage.yaml -o local-storage.yaml

kubectl apply -f local-storage.yaml
kubectl get storageclass
NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   
local-path   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  
```

### MetalLB

> <https://metallb.universe.tf/>  
> <https://metallb.universe.tf/configuration/>  
> <https://github.com/metallb/metallb>  

```sh
curl https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml -o metallb.yaml                                        
```

MetalLB 에서 구성할 수 있는 모드로는 다음 2가지

- Layer 2 모드  
- Layer 3 모드(BGP 모드)  


```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-service
  namespace: hello
#  annotations:
#    metallb.universe.tf/address-pool: first-pool
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: hello-pod
  type: LoadBalancer

```

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.10.XXX/32
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

설치 완료했다면 기존 ingress-controller 의 NodePort 로 운영하던 서비스를 LoadBalancer 로 변경  

기본적으로 LB IP 하나를 여러개의 LoadBalacner Service 가 같이 사용할 수 없다.  

서비스에 `metallb.universe.tf/allow-shared-ip` 주석을 추가하여 선택적 IP 공유를 활성화할 수 있다.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.9.1
    metallb.universe.tf/allow-shared-ip: "my-lb-service"
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - appProtocol: http
    name: http
    port: 80
    # nodePort: 30080
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    port: 443
    # nodePort: 30443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: LoadBalancer
  loadBalancerIP: 192.168.10.XXX
```

## Harbor

> <https://goharbor.io/>  
> <https://engineering.linecorp.com/ko/blog/harbor-for-private-docker-registry>

```shell
helm repo add harbor https://helm.goharbor.io

# 압축파일 다운로드, harbor-1.13.0.tgz 버전 설치됨
helm fetch harbor/harbor

# 압축 파일 해제
tar zxvf harbor-*.tgz
mv harbor harbor-helm
```

`local-path` `storageClass` 사용하도록 설정수정  

```yaml
persistence:
  enabled: true
  resourcePolicy: "keep"
  persistentVolumeClaim:
    registry:
      existingClaim: ""
      storageClass: "local-path"
      ...
```

`nginx ingress` 환경에서 동작하도록 설정수정  

```yaml
# values.yaml
  ingress:
    hosts:
      core: core.harbor.domain
    ...
    annotations:
      # note different ingress controllers may require a different ssl-redirect annotation
      # for Envoy, use ingress.kubernetes.io/force-ssl-redirect: "true" and remove the nginx lines below
      kubernetes.io/ingress.class: "nginx"
# proxy 환경에서 구성된다면 설정필요
externalURL: https://core.harbor.domain
```

ingress 에서 사용할 tls 10년짜리 인증서를 생성해서 사용

```sh
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=KR/ST=Seoul/L=Seoul/O=hello/OU=kouzie/CN=core.harbor.domain" \
 -key ca.key \
 -out ca.crt

TLS_CRT=$(cat ca.crt | base64) \
TLS_KEY=$(cat ca.key | base64) \
envsubst < harbor-ca-secret.yaml | \
kubectl apply -f -
```

```yaml
# values.yaml
expose:
  type: ingress
  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: "harbor-ca" # 위에서 생서한 secret 이름으로 설정
...
caSecretName: "harbor-ca" # 위에서 생서한 secret 이름으로 설정
```

```shell
cd harbor-helm
# namespace 생성
kubectl create ns harbor
helm install harbor -f values.yaml . -n harbor

watch kubectl get pods -n harbor
```

### k8s cluster 에 harbor dns 등록  

`k8s 노드` 의 `hosts` 파일에 `core.harbor.domain` 도메인을 등록.  

```conf
# /etc/hosts
192.168.10.XXX core.harbor.domain
```

`core.harbor.domain` 도메인을 `k8s corDNS` 에 설정

```sh
kubectl edit configmap coredns -n kube-system
```

```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        hosts {
          192.168.10.XXX    core.harbor.domain
          fallthrough
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
...
```

### image push & pull

`로컬 PC`, `k8s 노드` `docker` `daemon.json` 수정  

```json
// ubuntu: /etc/docker/daemon.json
// mac: ~/.docker/daemon.json
// 설정후 docker 데몬 재시작 필요
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "insecure-registries": [
    "https://core.harbor.domain"
  ]
}
```

설정 완료 후 login 및 이미지 push 확인  

```sh
docker login -u admin -p Harbor12345 core.harbor.domain

docker build -t hello:demo . 
docker tag hello:demo core.harbor.domain/library/hello:demo
docker push core.harbor.domain/library/hello:demo
```

## jenkins

> <https://github.com/jenkinsci/helm-charts>

```sh
helm repo add jenkins https://charts.jenkins.io

# 압축파일 다운로드, jenkins-4.7.2.tgz 버전 설치됨
helm fetch jenkins/jenkins

# 압축 파일 해제
tar zxvf jenkins-*.tgz
mv jenkins jenkins-helm
```

`Ingress, StoreClass` 설정 수정.  

```yaml
  ingress:
    enabled: true
    paths: []
    apiVersion: "networking.k8s.io/v1"
    labels: {}
    annotations: {}
    ingressClassName: nginx
    hostName: jenkins.cluster.local
    tls:
    # - secretName: jenkins.cluster.local
    #   hosts:
    #     - jenkins.cluster.local

persistence:
  enabled: true
  existingClaim:
  storageClass: "local-path"

jenkinsUrl: "https://jenkins.cluster.local"
```

```shell
cd jenkins-helm
# namespace 생성
kubectl create ns jenkins
helm install jenkins -f values.yaml . -n jenkins
# 비밀번호 확인
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo

watch kubectl get pods -n jenkins
```

### jenkins git SSL 무시

> <https://stackoverflow.com/questions/41930608/jenkins-git-integration-how-to-disable-ssl-certificate-validation>

사내 git 서버를 사용중이고 `unknown 서명된 인증서` 를 사용중이라면 아래와 같은 오류문구가 뜰 수 있다.  

```
SSL certificate problem: self signed certificate in certificate chain
```

`master, agent` 모두 `git ssl` 을 무시하는 환경변수 설정.  

```yaml
# master ssl disable
controller:
  ...
  initContainerEnv:
    - name: "GIT_SSL_NO_VERIFY"
      value: "true"
  containerEnv:
    - name: "GIT_SSL_NO_VERIFY"
      value: "true"

...

# agent ssl disable
agent:
  enabled: true
  ...
  # Pod-wide environment, these vars are visible to any container in the agent pod
  envVars: 
  - name: "GIT_SSL_NO_VERIFY"
    value: "true"
```

`PotTemplate` 과 같은 `jenkins pipeline` 문법을 사용할 경우 `values.yaml` 에서 설정한 환경변수가 `agent` 에서 동작하지 않기 때문에 아래와 같이 `Jenkinsfile` 에서 직접 환경변수 지정하는것을 권장  

```groovy
podTemplate(
  yaml: '''
kind: Pod
  ...
''',
  envVars: [envVar(key: 'GIT_SSL_NO_VERIFY', value: 'false')],
) {
    node(POD_LABEL) {
    }
}
```

### jenkins docker login secret  

`jenkins` 에서 `Harbor registry` 에 로그인 하기 위한 `login secret` 설정.  

```yaml
# harber-jenkins-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
  namespace: jenkins
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: $HARBOR_DOCKER_CONFIG_JSON
```

```sh
HARBOR_DOCKER_AUTH=$(echo -n 'admin:Harbor12345' | base64) \
HARBOR_DOCKER_CONFIG_JSON=$(echo -n '{"auths": {"core.harbor.domain": {"auth": "'$HARBOR_DOCKER_AUTH'"}}}' | base64) \
envsubst < harber-jenkins-secret.yaml | \
kubectl apply -f -
```

## argoCd

```shell
kubectl create namespace argocd

helm repo add argo https://argoproj.github.io/argo-helm

# 압축파일 다운로드, argo-cd-5.46.8.tgz 다운도르됨
helm fetch argo/argo-cd

# 압축 파일 해제
tar zxvf argo-cd-*.tgz
mv argo-cd argo-cd-helm
```

마찬가지로 `ingress` 를 통해 접근함으로 아래처럼 `value.yaml` 수정  

```yaml
server:
  ...
  ingress:
    # -- Enable an ingress resource for the Argo CD server
    enabled: true
    # -- Additional ingress annotations
    annotations: {
      kubernetes.io/ingress.class: nginx
    }
    ```
    
    ...
    # https redirect 방지
    server.insecure: true
...
```

```shell
cd argo-cd-helm
# namespace 생성
kubectl create ns argocd
helm install argocd -f values.yaml . -n argocd

# 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

watch kubectl get pods -n jenkins
```

## minio

> <https://github.com/minio/minio/tree/master/helm/minio>  
> k8s v1.28 버전 기준으로 <https://helm.min.io/> 의 helm 이 동작하지 않음.  
> 지원 중단된 문법이 많아 별도의 CRD 를 설치하지 않는이상 동작하지 않는다.  
>
> 위 github 링크 참조하여 helm 으로 minio 설치  

```shell
kubectl create ns minio
helm repo add minio https://charts.min.io/

# 압축파일 다운로드, minio-5.0.14.tgz 다운도르됨
helm fetch minio/minio

# 압축 파일 해제
tar zxvf minio-*.tgz
mv minio minio-helm
```

마찬가지로 ingress 를 통해 접근함으로 아래처럼 `value.yaml` 수정  

```yaml
ingress:
  enabled: true
  ingressClassName: nginx
  labels: {}
...

consoleIngress:
  enabled: true
  ingressClassName: nginx
  labels: {}
...
```

`StorageClass` 는 `local-path` 로 수정

```yaml
storageClass: "local-path"
volumeName: ""
accessMode: ReadWriteOnce
size: 100Gi
```

그 외의 설정들도 toy 세팅으로 수정

```conf
resources.requests.memory=512Mi 
replicas=1 
persistence.enabled=false 
mode=standalone 
rootUser=rootuser,rootPassword=rootpass123
```

```shell
cd minio-helm
# namespace 생성
kubectl create ns minio
helm install minio -f values.yaml . -n minio
```


## rabbitmq

mqtt 플러그인이 설치되어 있는 rabitmq 설치 및 배포

```shell
cd dev-tool/rabbitmq
docker build -t rabbitmq_mqtt ./docker --platform=linux/amd64
docker tag rabbitmq_mqtt core.harbor.domain/library/rabbitmq_mqtt  
docker push core.harbor.domain/library/rabbitmq_mqtt
```

```sh
kubectl create ns rabbitmq-ns
HARBOR_DOCKER_AUTH=$(echo -n 'admin:Harbor12345' | base64) \
HARBOR_DOCKER_CONFIG_JSON=$(echo -n '{"auths": {"core.harbor.domain": {"auth": "'$HARBOR_DOCKER_AUTH'"}}}' | base64) \
envsubst < harber-jenkins-secret.yaml | \
kubectl apply -f -

kubectl apply -f rabbitmq-pvc.yaml 
kubectl apply -f rabbitmq-state.yaml
```

## ELK

### elastic search

> <https://github.com/elastic/helm-charts>  
> 위 github 에서 elastic search, kibana, logstash, microbeat 에 해당하는 helm 차트 설치 가능  

```shell
# elastic search 설치
kubectl create ns es
helm repo add elastic https://helm.elastic.co

# 압축파일 다운로드, elasticsearch-8.5.1.tgz 다운도르됨
helm fetch elastic/elasticsearch

# 압축 파일 해제
tar zxvf elasticsearch-*.tgz
mv elasticsearch elasticsearch-helm
```

`ingress` 를 통해 외부에서 접근할 수 있도록 아래와 도메인과 `ingress.enable` 설정  

그리고 `elastic search` 의 버전이 올라가면서 기본 `https` 통신을 요구함으로  
제공되는 `ingress` 를 통해 `https` 로 접근하도록 어노테이션을 추가한다.  

```yaml

# Enabling this will publicly expose your Elasticsearch instance.
# Only enable this if you have security enabled on your cluster
ingress:
  enabled: true
  annotations: 
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    nginx.ingress.kubernetes.io/secure-backends: "true"
    ingress.kubernetes.io/ssl-passthrough: "true"
  kubernetes.io/ingress.class: nginx
  # kubernetes.io/tls-acme: "true"
  className: "nginx"
  pathtype: ImplementationSpecific
  hosts:
    - host: chart-example.es.local
      paths:
        - path: /
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local
```

`StorageClass` 는 `local-path` 사용하도록 설정

```yaml
volumeClaimTemplate:
  storageClassName: local-path
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 10Gi
```

```shell
helm install elasticsearch -f values.yaml . -n es

# 1. Watch all cluster members come up.
#   $ kubectl get pods --namespace=es -l app=elasticsearch-master -w
# 2. Retrieve elastic user's password.
#   $ kubectl get secrets --namespace=es elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
# 3. Test cluster health using Helm test.
#   $ helm --namespace=es test elasticsearch
```

### kibana  

UI 접근을 위해 `kibana` 까지는 설치 진행  

> <https://github.com/elastic/helm-charts/tree/main/kibana>

```shell
# kibana 설치
kubectl create ns es
helm repo add elastic https://helm.elastic.co

# 압축파일 다운로드, kibana-8.5.1.tgz 다운도르됨
helm fetch elastic/kibana

# 압축 파일 해제
tar zxvf kibana-8.5.1.tgz
mv kibana kibana-helm
```

`kibana` 의 `ingress` 설정한 후 배포하면 된다.  

```yaml
ingress:
  enabled: true
  className: "nginx"
  pathtype: ImplementationSpecific
  annotations: {}
  # kubernetes.io/ingress.class: nginx
  # kubernetes.io/tls-acme: "true"
  hosts:
    - host: kibana-example.es.local
      paths:
        - path: /
  #tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local
```

```
helm install kibana -f values.yaml . -n es
```