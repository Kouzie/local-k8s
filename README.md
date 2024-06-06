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
  name: ingress-nginx-controller
  namespace: ingress-nginx
  annotations:
    metallb.universe.tf/allow-shared-ip: my-lb-service
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

host 파일에 아래 domain 입력 후 접속

<https://core.harbor.domain/>
admin/Harbor12345

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

`Ingress, StoreClass, agent` 설정 수정.  

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

agent:
  ...
  # image: "jenkins/inbound-agent"
  # tag: "3107.v665000b_51092-15"
  image: "core.harbor.domain/library/jenkins/inbound-agent"
  tag: "aws-cli"
  ...
  # You may want to change this to true while testing a new image
  alwaysPullImage: false
  ...
```

```shell
cd jenkins-helm
# namespace 생성
kubectl create ns jenkins
helm install jenkins -f values.yaml . -n jenkins
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

<https://jenkins.cluster.local/>
admin/...

```sh
# 비밀번호 확인
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
```

### aws cli 용 jenkins 이미지 생성

pipeline 에서 사용할 각종 client 툴을 설치한 상태로 jenkins agent 를 실행하기 위해 커스텀 이미지 생성

```
docker build --platform linux/amd64 -t jenkins_aws_dockerfile -f jenkins_aws_dockerfile .
docker tag jenkins_aws_dockerfile core.harbor.domain/library/jenkins/inbound-agent:aws-cli
docker push core.harbor.domain/library/jenkins/inbound-agent:aws-cli
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

watch kubectl get pods -n jenkins
```

<https://argocd.example.com>
admin/...

```sh
# 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## minio

> <https://github.com/minio/minio/tree/master/helm/minio>  
> k8s v1.28 버전 기준으로 <https://helm.min.io/> 의 helm 이 동작하지 않음.  
> 지원 중단된 문법이 많아 별도의 CRD 를 설치하지 않는이상 동작하지 않는다.  
>
> 위 github 링크 참조하여 helm 으로 minio 설치  

먼저 minio 에서 사용할 minio-pv 생성

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: rancher.io/local-path
  finalizers:
  - kubernetes.io/pv-protection
  name: minio-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 100Gi
  hostPath:
    path: /home/k8s-storage/minio-pv
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - beylesswsg
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-path
  volumeMode: Filesystem
status:
  phase: Bound
```

기존 배포되어있는 pv 를 참고해서 만들면 쉽게 만들 수 있다.

```sh
kubectl get pv <pvnmae> -o yaml
```

문제가 발생하면 생성되었던 pv, pvc 를 삭제하고 다시 만들면 됨(directory 가 삭제되지 않음)

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
volumeName: "minio-pv" # 위에서 설정한 pv name 지정
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

호스트파일에 아래 domain 을 입력하고 접속

<https://console.minio-example.local/>
rootuser/rootpass123

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

아래 domain host 파일에 입력 후 접속  

<https://kibana-example.es.local/app/home>
elastic/password

## monitoring

- OTEL  
- LGTM
  - Loki, like Prometheus, but for logs.
  - Grafana, the open and composable observability and data visualization platform.
  - Tempo, a high volume, minimal dependency distributed tracing backend.
  - Mimir, the most scalable Prometheus backend.

### OTEL Collector(Daemon)

> <https://github.com/open-telemetry/opentelemetry-helm-charts>

```shell
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts

# 압축파일 다운로드, opentelemetry-collector-0.78.1.tgz 버전 설치됨
helm fetch open-telemetry/opentelemetry-collector

# 압축 파일 해제
tar zxvf opentelemetry-collector-*.tgz
mv opentelemetry-collector opentelemetry-collector-helm
```

```yaml
# Valid values are "daemonset", "deployment", and "statefulset".
mode: "deployment"

# Specify which namespace should be used to deploy the resources into
namespaceOverride: "monitoring"

presets:
  # Configures the collector to collect logs.
  # Adds the filelog receiver to the logs pipeline
  # and adds the necessary volumes and volume mounts.
  # Best used with mode = daemonset.
  # See https://opentelemetry.io/docs/kubernetes/collector/components/#filelog-receiver for details on the receiver.
  logsCollection:
    enabled: false # 직접 로그를 전달하는 것만 기록
    includeCollectorLogs: false

service:
  # Enable the creation of a Service.
  # By default, it's enabled on mode != daemonset.
  # However, to enable it on mode = daemonset, its creation must be explicitly enabled
  enabled: true

  type: ClusterIP
```


```sh
kubectl create ns monitoring
helm install opentelemetry-collecor -f values.yaml . -n monitoring
```

### OTEL Colletor(Sidecar)

`sidecar` 방식으로 `OTEL 컬렉터` 설치

> <https://github.com/open-telemetry/opentelemetry-operator>  
> <https://opentelemetry.io/docs/kubernetes/operator/>  
> <https://cert-manager.io/docs/installation/helm/>

```shell
# cert-manager 설치
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.2/cert-manager.yaml
kubectl get all -n cert-manager

# opentelemetry-operator 설치
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
kubectl get all -n opentelemetry-operator-system
```

`sidecar.opentelemetry.io/inject` 라벨이 설정된 pod 에 sidecar 가 같이 실행됨.  


### Loki

> <https://grafana.com/docs/loki/latest/setup/install/helm/>  
> <https://grafana.com/docs/loki/latest/setup/install/helm/install-scalable>

```shell
helm repo add grafana https://grafana.github.io/helm-charts
helm search repo grafana

# 압축파일 다운로드, loki-5.42.0.tgz 버전 설치됨
helm fetch grafana/loki

# 압축 파일 해제
tar zxvf loki-*.tgz
mv loki loki-helm
```


```yaml
# self-monitoring 미사용  
test:
  # ...
  enabled: false

monitoring:
  # ...
  selfMonitoring:
    enabled: false

loki:
  # ...
  # Should authentication be enabled
  auth_enabled: false
  # ...
  storage:
    bucketNames:
      chunks: chunks
      ruler: ruler
      admin: admin
    type: s3
    s3:
      s3: null
      endpoint: http://minio.minio.svc.cluster.local:9000/loki
      region: null
      secretAccessKey: rootpass123
      accessKeyId: rootuser
      signatureVersion: null
      s3ForcePathStyle: true
      insecure: true
      http_config: {}
  commonConfig:
    replication_factor: 1 # replica 개수대로 동작하지 않으면 로그수집을 진행하지 않음.

# 각 서비스별 replicas 수는 모두 1로 고정  
read:
  # ...
  replicas: 1
  persistence:
    # -- Enable StatefulSetAutoDeletePVC feature
    enableStatefulSetAutoDeletePVC: false
    
write:
  # ...
  replicas: 1
  persistence:
    # -- Enable volume claims in pod spec
    volumeClaimsEnabled: false

backend:
  # ...
  replicas: 1      
  persistence:
    # -- Enable volume claims in pod spec
    volumeClaimsEnabled: false
```

mionio 를 s3 대신 사용할 경우 위와같이 uri 기반으로 bucket 이름을 설정해야함.  

```sh
kubectl create namepsace loki
helm install loki -f values.yaml . -n loki
```

### Grafana

> <https://github.com/grafana/helm-charts>

```shell
helm repo add grafana https://grafana.github.io/helm-charts
helm search repo grafana

# 압축파일 다운로드, grafana-7.2.5.tgz 버전 설치됨
helm fetch grafana/grafana

# 압축 파일 해제
tar zxvf grafana-*.tgz
mv grafana grafana-helm
```

```yaml
# ingress 설정
ingress:
  enabled: true
  annotations: 
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
  labels: {}
  path: /

# storage class pvc 설정
persistence:
  type: pvc
  enabled: true
  storageClassName: "local-path"
```

```sh
helm install grafana -f values.yaml . -n monitoring
```

```sh
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo 
# strongpassword
```

chart-example.local 를 hosts 파일에 등록 후 접속

### Thanos

> <https://grafana.com/docs/mimir/latest/>  
> <https://grafana.com/docs/helm-charts/mimir-distributed/latest/get-started-helm-charts/>
>
> <https://github.com/thanos-io/thanos>  
> <https://thanos.io/tip/thanos/quick-tutorial.md/>  

원래는 Mimir 설치 예정이었으나 부족한 문서와 환경으로 인해 thanos 설치로 변경  

```shell
kubectl create namespace thanos

helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo bitnami

# 압축파일 다운로드, thanos-12.23.0.tgz 설치됨
helm fetch bitnami/thanos

# 압축 파일 해제
tar zxvf thanos-*.tgz
mv thanos thanos-helm
```

s3 에 접근하기 위한 secret 설정을 해야 `[compact, ruler, store]` 컴포넌트 사용이 가능하다. 

```yaml
# objstore.yml
type: s3
config:
  bucket: thanos
  endpoint: minio.minio.svc.cluster.local:9000
  access_key: rootuser
  secret_key: rootpass123
  insecure: true
```

```sh
kubectl create secret generic thanos-objstore-secret --from-file=objstore.yml -n prometheus
kubectl create secret generic thanos-objstore-secret --from-file=objstore.yml -n thanos
```

사이드카 방식을 사용함으로 receive 컴포넌트를 제외한 모든 컴포넌트를 `enable: true` 로 설정  
prometheus 의 thanos sidecar, alert manager 연동을 진행.  

```sh
existingObjstoreSecret: "thanos-objstore-secret"
## @param existingObjstoreSecretItems Optional item list for specifying a custom Secret key. If so, path should be objstore.yml

query:
  # ...
  stores: 
  # prometheus thanos sidecar service name
  - prometheus-kube-prometheus-thanos-discovery.prometheus.svc.cluster.local:10901


queryFrontend:
  enabled: true
  # ...
  config:
    type: IN-MEMORY
    config:
      max_size: 512MB
      max_size_items: 100
      validity: 120s

compactor:
  enabled: true
  # ...
  retentionResolutionRaw: 30d
  retentionResolution5m: 30d
  retentionResolution1h: 1y # 10y is too long
  persistence:
    ## @param compactor.persistence.enabled Enable data persistence using PVC(s) on Thanos Compactor pods
    ##
    enabled: false

storegateway:
  enabled: true
  # ...
  config:
    type: IN-MEMORY
    config:
      max_size: 300MB
      max_item_size: 120MB
  persistence:
    ## @param storegateway.persistence.enabled Enable data persistence using PVC(s) on Thanos Store Gateway pods
    ##
    enabled: false

ruler:
  enabled: true
  # ...
  alertmanagers: 
    - http://prometheus-kube-prometheus-alertmanager.prometheus.svc.cluster.local:9093
  config:
    groups:
      - name: "metamonitoring"
        rules:
          - : "PrometheusDown"
            expr: absent(up{prometheus="monitoring/prometheus-operator"})
  persistence:
    ## @param ruler.persistence.enabled Enable data persistence using PVC(s) on Thanos Ruler pods
    ##
    enabled: false
```

### prometheus-community

thanos 는 prometheus 를 기반으로 동작하기 때문에 서로 의존관계이다.  
prometheus 에서도 thanos 연동을 위한 설정을 해줘야한다.  

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm search repo prometheus-community

# 압축파일 다운로드, kube-prometheus-stack-56.4.0.tgz 버전 설치됨
helm fetch prometheus-community/kube-prometheus-stack

# 압축 파일 해제
tar zxvf kube-prometheus-stack-*.tgz
mv kube-prometheus-stack kube-prometheus-stack-helm
```

`Thanos Sidecar` 에서 메트릭 데이터를 직접 `ObjectStorage` 에 넣기 때문에 연결 설정이 필요하다.  

```yaml
# objstore.yml
type: s3
config:
  bucket: thanos
  endpoint: minio.minio.svc.cluster.local:9000
  access_key: rootuser
  secret_key: rootpass123
  insecure: true
```

```sh
kubectl create secret generic thanos-objstore-secret --from-file=objstore.yml -n prometheus
```

```yaml
# values.yaml
prometheus:
  prometheusSpec:
    # ...
    enableRemoteWriteReceiver: true # 외부에서 들어오는 remote_write를 허용
    # ...
    thanos:
      objectStorageConfig:
        existingSecret: 
          name: "thanos-objstore-secret"
          key: "objstore.yml"
  thanosService:
    enabled: true
```

위와 같이 설정후 실행시키면 `Proemetheus` 와 `Thanos Sidecar` 가 같이 실행된다.  

```sh
kubectl create namespace prometheus
helm install prometheus -f values.yaml . -n prometheus

kubectl get pod/prometheus-prometheus-kube-prometheus-prometheus-0 -n prometheus  -o=jsonpath='{.spec.containers[*].name}' | tr ' ' '\n'
# prometheus
# config-reloader
# thanos-sidecar
```

### Tempo

> <https://grafana.com/docs/tempo/latest/setup/helm-chart/>

msa 형태로 운영되는 tempo-distributed, 단일 서버로 운영되는 monolithic helm 차트 지원

여기선 monolithic 방식을 사용한다.  

```shell
helm repo add grafana https://grafana.github.io/helm-charts
helm search repo grafana

# 압축파일 다운로드, tempo-1.7.1.tgz 버전 설치됨, 모놀리식 버전
helm fetch grafana/tempo

# 압축 파일 해제
tar zxvf tempo-*.tgz
mv tempo tempo-helm
```

모놀리식 운영방식인 만큼 persistence 와 같은 추가설정 없이 동작되도록 되어있다.  

`service graph` 작성을 위한 `prometheus remote write` 과 `s3` 에 대한 설정 지행.

```yaml
tempo:
  # ...
  metricsGenerator:
  # -- If true, enables Tempo's metrics generator (https://grafana.com/docs/tempo/next/metrics-generator/)
  enabled: true
  remoteWriteUrl: "http://prometheus-kube-prometheus-prometheus.prometheus.svc.cluster.local:9090/api/v1/write"
  # ...
  storage:
    trace:
      # tempo storage backend
      # refer https://grafana.com/docs/tempo/latest/configuration/
      ## Use s3 for example
      backend: s3
      # store traces in s3
      s3:
        bucket: tempo                                   # store traces in this bucket
        endpoint: minio.minio.svc.cluster.local:9000  # api endpoint
        access_key: rootuser                                 # optional. access key when using static credentials.
        secret_key: rootpass123                                 # optional. secret key when using static credentials.
        insecure: true                                 # optional. enable if endpoint is http
      # backend: local
```

```sh
kubectl create namespace tempo
heln install tempo -f values.yaml . -n tempo
```

### fluentbit

> <https://docs.fluentbit.io/manual/installation/kubernetes>
> <https://github.com/fluent/helm-charts>  

```sh
helm repo add fluent https://fluent.github.io/helm-charts
helm search repo fluent

# 압축파일 다운로드, fluent-bit-0.43.0.tgz 버전 설치됨
helm fetch fluent/fluent-bit
# 압축 파일 해제
tar zxvf fluent-bit-*.tgz
mv fluent-bit fluent-bit-helm
```

읽어드릴 file, 그리고 OTEL 컬렉터로 전달  

```sh
heln install fluent-bit -f values.yaml . -n monitoring
```

로그파일이 많다면 `too many open files` 에러가 출력될 수 있음.  
단일 계정 `file descriptor` 사용량 제한에 걸렸기 때문.  

사용량을 영구적으로 늘리기 위해 `/etc/sysctl.conf` 에서 아래 문자열 추가

```sh
sudo vi /etc/sysctl.conf

# sysctl.conf 에 해당 문자열 추가, default 는 256
fs.inotify.max_user_instances=8192

# 적용
sudo sysctl -p
```

## istio

```sh
helm repo add istio https://istio-release.storage.googleapis.com/charts
# 압축파일 다운로드, base-1.20.3.tgz 버전 설치됨
helm fetch istio/base
# 압축파일 다운로드, istiod-1.20.3.tgz 버전 설치됨
helm fetch istio/istiod

# 압축 파일 해제
tar zxvf base-*.tgz
mv base base-helm
tar zxvf istiod-*.tgz
mv istiod istiod-helm

kubectl create namespace istio-system
```

`istio k8s CRD` 설치

```sh
# base-helm
helm install istio-base -f values.yaml . -n istio-system

kubectl get crd 
# authorizationpolicies.security.istio.io
# envoyfilters.networking.istio.io
# istiooperators.install.istio.io
# peerauthentications.security.istio.io
# proxyconfigs.networking.istio.io
# requestauthentications.security.istio.io
# serviceentries.networking.istio.io
# sidecars.networking.istio.io
# telemetries.telemetry.istio.io
# virtualservices.networking.istio.io
# wasmplugins.extensions.istio.io
# ...
```

`Istio deamon chart` 설치

```sh
# istiod-helm
helm install istiod -f values.yaml . -n istio-system

kubectl get all -n istio-system                     
# NAME                         READY   STATUS    RESTARTS   AGE
# pod/istiod-bc4584967-pvpgv   1/1     Running   0          6m26s

# NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                 AGE
# service/istiod   ClusterIP   10.97.153.154   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP   6m26s

# NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/istiod   1/1     1            1           6m26s

# NAME                               DESIRED   CURRENT   READY   AGE
# replicaset.apps/istiod-bc4584967   1         1         1       6m26s

# NAME                                         REFERENCE           TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
# horizontalpodautoscaler.autoscaling/istiod   Deployment/istiod   <unknown>/80%   1         5         1          6m26s
```

```sh
helm ls -n istio-system
# NAME            NAMESPACE       REVISION   ...  STATUS    ...  APP VERSION
# istio-base      istio-system    1          ...  deployed  ...  1.20.3     
# istiod          istio-system    1          ...  deployed  ...  1.20.3     
```

### instio ingress gateway controller 설치  

> <https://istio.io/latest/docs/setup/additional-setup/gateway/>

위 url 에서 요구한대로 `Kubernetes YAML` 기반으로 `LoadBalancer, Deployment, Role, RoleBinding` 생성

단 LoadBalancer 의 경우 이미 nginx-ingress 가 80, 443 포트를 사용중임으로  
istio-ingress 는 8080,8443 을 사용하도록 설정.

그리고 MetalLB 를 사용중임으로 아래 annotation 지정 필요.

```yaml
# ingress.yaml
# ...
metadata:
  name: istio-ingressgateway
  namespace: istio-ingress
  annotations:
    metallb.universe.tf/allow-shared-ip: "my-lb-service"
# ...
```

```sh
kubectl create namespace istio-ingress
kubectl apply -f ingress.yaml
```

### 데모서비스 실행

> <https://istio.io/latest/docs/examples/bookinfo/>

`book-demo` `namespace` 에 `istio-injection` 라벨을 설정하고 실행.
생성되는 `book-demo` `namespace` 에서 생성되는 pod 에 istio sidecar 가 같이 실행되는지 확인  

```sh
kubectl create namespace book-demo
kubectl label namespace book-demo istio-injection=enabled
kubectl apply -f bookinfo.yaml -n book-demo
kubectl get all -n book-demo
# NAME                                 READY   STATUS    RESTARTS   AGE
# pod/details-v1-698d88b-6ppfv         2/2     Running   0          42s
# pod/productpage-v1-675fc69cf-lghct   2/2     Running   0          42s
# pod/ratings-v1-6484c4d9bb-7bt57      2/2     Running   0          42s
# pod/reviews-v1-5b5d6494f4-8h8fz      2/2     Running   0          42s
# ...

# NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
# service/details       ClusterIP   10.105.228.184   <none>        9080/TCP   42s
# service/productpage   ClusterIP   10.99.64.113     <none>        9080/TCP   42s
# service/ratings       ClusterIP   10.98.143.228    <none>        9080/TCP   42s
# service/reviews       ClusterIP   10.101.1.40      <none>        9080/TCP   42s

# NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/details-v1       1/1     1            1           42s
# deployment.apps/productpage-v1   1/1     1            1           42s
# deployment.apps/ratings-v1       1/1     1            1           42s
# deployment.apps/reviews-v1       1/1     1            1           42s
# ...

# NAME                                       DESIRED   CURRENT   READY   AGE
# replicaset.apps/details-v1-698d88b         1         1         1       42s
# replicaset.apps/productpage-v1-675fc69cf   1         1         1       42s
# replicaset.apps/ratings-v1-6484c4d9bb      1         1         1       42s
# replicaset.apps/reviews-v1-5b5d6494f4      1         1         1       42s
# ...
```

파드에서 실행중인 컨테이너 이름 출력

```sh
kubectl get pod/details-v1-698d88b-6ppfv -n book-demo -o=jsonpath='{.spec.containers[*].name}' | tr ' ' '\n'

# details
# istio-proxy
```

### 관측 백엔드 통합  

> <https://istio.io/latest/docs/ops/integrations/prometheus/>  
> <https://github.com/istio/istio/tree/master/samples/addons/>  

`istio` 에서 제공하는 `addon` 을 통해 `[prometheus, jeager, grafana]` 등을 설치할 수 있다.  

기존에 설치해둔 `prometheus` 와 `tempo` 가 있음으로 `addon` 으로 설치하지 않고 추가 설정을 통해 통합을 진행한다.  

기존에 설치했던 `prometheus` 의 `additionalScrapeConfigs` 에 아래와 같이 istio 관측 데이터를 수집할 수 있도록 구성.  

```yaml
prometheus:
  ...
  prometheusSpec:
  ...
    additionalScrapeConfigs:
      - job_name: 'istiod' # control plan 수집
        kubernetes_sd_configs:
        - role: endpoints
          namespaces:
            names:
            - istio-system
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: istiod;http-monitoring
      - job_name: 'envoy-stats' # data plan 수집
        metrics_path: /stats/prometheus
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_container_port_name]
          action: keep
          regex: '.*-envoy-prom'
```

실제 istio 데모로 올렸던 `pod` 의 `ports.name` 에 `regex` 에 맞는 문자열이 설정되어있다.  

```sh
kubectl get pod details-v1-698d88b-6ppfv -o=jsonpath='{.spec.containers[*].ports[*].name}' -n book-demo 
# http-envoy-prom
```

`envoy` 에서 관측데이터 전달을 `zipkin b3` 방식만 사용 가능함으로, `tempo-helm` 의 `values.yaml` 에 `zipkin` 프로토콜 수신지 설정

```yaml
tempo:
  ...
  receiver:
  ...
    zipkin:
      endpoint: "0.0.0.0:9411"
```

`envoy` 에서 `trace` 관측데이터를 `zipkin` 이나 `jeager` 로 전달하도록 `defaultConfig` 를 수정

```yaml
meshConfig:
  enablePrometheusMerge: true
  enableTracing: true
  defaultConfig: # envoy default config
    tracing:
      zipkin:
        address: tempo.tempo.svc.cluster.local:9411
      sampling: 100.0
```

이제 모든 `envoy` 에서 위 `address` 로 `trace` 데이터를 전달한다.  

### kiali

```sh
helm repo add istio https://kiali.org/helm-charts

helm search repo kiali

# 압축파일 다운로드, kiali-server-1.80.0.tgz 버전 설치됨
helm fetch kiali/kiali-server

# 압축 파일 해제
tar zxvf kiali-server-*.tgz
mv kiali-server kiali-server-helm
```

`kiali` 의 관측데이터는 `prometheus` 를 통해 표시 및 계산된다. 기존에 생성해둔 `thanos` 기반 `prometheus` 쿼리기를 사용

```yaml
istio_namespace: "istio-system" # default is where Kiali is installed

auth:
  openid: {}
  openshift: {}
  strategy: "anonymous"
...
external_services:
  custom_dashboards:
    enabled: true
  istio:
    root_namespace: ""
  prometheus:
    url: "http://thanos-query-frontend.thanos.svc.cluster.local:9090/"
  tracing:
    enabled: true
    in_cluster_url: "http://tempo.tempo.svc.cluster.local:3100/"
    provider: "tempo"
    use_grpc: false
...
server:
  port: 20001
  observability:
    metrics:
      enabled: true
      port: 9090
  web_root: "/dashboards/kiali"
  web_fqdn: kiali.istio.local

```

```sh
# base-helm
helm install kiali-server -f values.yaml . -n istio-system
```

`kiali.istio.local` hosts 파일에 등록 후 접속

<https://kiali.istio.local/dashboards/kiali> 접속
