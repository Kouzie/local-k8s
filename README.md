
## host file 참고

nginx ingress 를 통해 접속할 수 있도록 `로컬 PC` `hosts` 파일 수정  

```
192.168.10.XXX  core.harbor.domain jenkins.cluster.local argocd.example.com
```

### 접속방법

<https://core.harbor.domain:30443/>
admin/Harbor12345

<https://jenkins.cluster.local:30443/>
admin/...

<https://argocd.example.com:30443>
admin/...

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
# 하지만 내부 클러스터에서 redirect 를 아래 url 로 하기 때문에 사용하지 말자.
# externalURL: https://core.harbor.domain:30443
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
    "https://core.harbor.domain:30443"
  ]
}
```

설정 완료 후 login 및 이미지 push 확인  

```sh
docker login -u admin -p Harbor12345 core.harbor.domain:30443

docker build -t hello:demo . 
docker tag hello:demo core.harbor.domain:30443/library/hello:demo
docker push core.harbor.domain:30443/library/hello:demo
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

jenkinsUrl: "https://jenkins.cluster.local:30443"
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
HARBOR_DOCKER_CONFIG_JSON=$(echo -n '{"auths": {"core.harbor.domain:30443": {"auth": "'$HARBOR_DOCKER_AUTH'"}}}' | base64) \
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
