---
title: "🖥️ 02. 로컬 K8S 클러스터 생성"
weight: 2
description: ""
icon: "article"
date: "2025-02-04T01:15:27+09:00"
lastmod: "2025-02-04T01:15:27+09:00"
draft: false
toc: true

---

---

## 1. What is Kind?

Kind 는 로컬 컴퓨터 환경에 쿠버네티스 클러스터를 손쉽고 빠르게 설치 하기 위해  만들어진 도구 입니다.

Kind는 Go 언어를 기반으로 만들어 졌으며, Docker 이미지를 기반으로 [kubeadm](https://github.com/kubernetes/kubeadm)을 이용하여 클러스터를 배포 합니다.

kind 공식 홈페이지 : [kind.sigs.k8s.io](https://kind.sigs.k8s.io)

kind와 유사하게 멀티노드 기반 쿠버네티스 로컬 클러스터 구축 도구에는 아래와 같은 것들이 있습니다.

| 도구명 | 공식 URL |
|------------------|-------------------|
| minikube | https://minikube.sigs.k8s.io |
| k3s | https://k3s.io |
| MicroK8s | https://microk8s.io |
| k3d | https://k3d.io |


## 2. Kind 설치 하기
설치 가이드 원본 URL : https://kind.sigs.k8s.io/docs/user/quick-start/#installation
### - MacOS
```bash
brew install kind
```

### - Windows

```bash
choco install kind
```

### - Linux

```bash
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.15.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\some-dir-in-your-PATH\kind.exe
```

### - Kubectx, Kubens 설치

```
choco install kubectx
choco install kubens
```

## 3. Kind 로 클러스터 생성 (초간단)

### - 클러스터 생성

```bash
kind create cluster # Default cluster context 이름은 'kind' 로 생성
kind create cluster --name dangtong # cluster context 이름을 'dangtong' 으로 지정
```

### - 클러스터 생성 확인

```bash
kind get clusters
kubectl cluster-info --context dangtong
```

### - 클러스터 삭제

```bash
kind delete cluster

kind delete clusters kind-local-cluster
```

## 4. 설정 파일을 이용한 Kind 클러스터 생성

### - 설정 파일을 이용한 클러스터 생성

설정파일을 이용해서 kind 클러스터를 생성할 수 있습니다.

```bash
kind create cluster --config kind-example-config.yaml
```

### - 3개 노드 클러스터 생성 예시

3개 노드(1 controller, 2worker) 클러스터 설정

```{
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

### - 6개 노드 클러스터 생성 예시

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
```

### - 쿠버네티스 버전 설정

쿠버네티스 버전에 따른 이미지는 링크에서 확인 가능 : https://github.com/kubernetes-sigs/kind/releases

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.16.4@sha256:b91a2c2317a000f3a783489dfb755064177dbc3a0b2f4147d50f04825d016f55
- role: worker
  image: kindest/node:v1.16.4@sha256:b91a2c2317a000f3a783489dfb755064177dbc3a0b2f4147d50f04825d016f55
```

### - 네트워크 설정

- Pod Subnet 설정

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  podSubnet: "10.244.0.0/16"
```

- Service Subnet 설정

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  serviceSubnet: "10.96.0.0/12"
```

- Default CNI 설정

Caliaco 완 같은 3rd party CNI 사용을 위해서는 default CNI 설치를 하지 말아야 합니다.

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  # default CNI가 설치 되지 않습니다.
  disableDefaultCNI: true
```

- kube-proxy 모드 설정

iptables 또는 IPVS 중에 선택해서 사용 가능. default 는 iptables

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  kubeProxyMode: "ipvs"
```

### - 최종 클러스터 생성

- 클러스터 yaml 작성
- 파일명 : 3-node-cluster.yml
- 노드 이미지 버전 참조 : https://github.com/kubernetes-sigs/kind/releases

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cwave-cluster
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  image: kindest/node:v1.32.5
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
  image: kindest/node:v1.32.5
- role: worker
  image: kindest/node:v1.32.5
networking:
  serviceSubnet: "10.120.0.0/16"
  podSubnet: "10.110.0.0/16"
```

- 클러스터 생성

```
# 생성
kind create cluster  --config ./3-node-cluster.yml

# 삭제
kind delete cluster --name cwave-cluster
```

- 클러스터 접속 정보 확인

```
kind get kubeconfig --internal --name cwave-cluster
```

- IDE 컨테이너에 Kind 네트워크 주석제거
```yml
name: "aws-cicd-practice"
services:
  code-server:
    image: dangtong76/cicd-devops-ide:arm64-v2 
    container_name: "ide"
    networks: # 네트워크 항목 주석제거
      - kind_network 
    environment:
      AUTH: none
      #FILE__PASSWORD: /run/secrets/code-server-password
    env_file:
      - .env
    working_dir: /code
    ports:
      - "8080:8080" # istory-web
      - "1314:1314" # hugo port 1
      - "1315:1315" # hugo port 2
      - "8444:8443" # vscode service port
      - "5500:5500"
    # secrets:s
    #   - code-server-password
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - devops-cicd-apps:/code/devops-cicd-apps
      - devops-cicd-vscode:/config
networks: # 네트워크 항목 주석제거
  kind_network:
    name: kind
    external: true
volumes:
  devops-cicd-apps:
    external: true
    name: devops-cicd-apps
  devops-cicd-vscode:
    external: true
    name: devops-cicd-vscode
```

- ~/.kube/config 파일의 API 접속 정보 수정
```bash
docker network inspect kind # cwave-cluster-control-plane 의 IP주소 확인

vi ~/.kube/config
```
아래와 같이 kubectl 접속 정보 수정
```yml
apiVersion: v1
clusters:
- cluster:
    server: https://<cwave-cluster-control-plane 의 IP주소>:6443
  name: kind-cwave-cluster
```
API 서버가 접속 되는지 확인하기
```bash
kubectl get no 

### 출력이 아래와 같은 형식으로 나오면 정상
root@85bed0161b08:~/.kube# kubectl get no
NAME                          STATUS   ROLES           AGE   VERSION
cwave-cluster-control-plane   Ready    control-plane   30m   v1.32.5
cwave-cluster-worker          Ready    <none>          30m   v1.32.5
cwave-cluster-worker2         Ready    <none>          30m   v1.32.5
```
## 5. MetalLB 설치

- MetalLB 설치

```
# kubectl 이용
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml

# helm 차트 이용
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb
```

- kind network의 IP 대역 확인
```
docker network inspect kind --format '{{(index .IPAM.Config 0).Subnet}}'
```

- MetalLB 설정

```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: cwave-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.247.100-192.168.247.110 # Docker kind 네트워크의 IP 대역으로 구간 할당
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: cwave-loadbalancer-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - cwave-pool
```

## 6. Ingress 및 LoadBalancer 설정

Ingress 및 Loadbalancer 를 설정하기 위해서는 KIND 를 이용한 클러스터 생성시  extraPortMapping 설정을 하고, kubeadm툴을 통해 custom node label 을 노드에 설정해야 합니다.

### - Ingress 가능한 클러스터 생성

kind 클러스터를 extraPortMappings 및 node-lables 설정과 함께 생성 합니다.

- ExtreaPortMappings : 로컬 호스트가 80 및 443 포트를 통해 Ingress Controller로 요청이 가능하게 설정합니다.
- node--labels : Ingress Controller 가 특정 라벨을 가진 노드에서만 수행 되도록 합니다.

```
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```

### - Countour Ingress 생성

- Contour 설치

```
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

- Contour 설정 업데이트 

```
kubectl patch daemonsets -n projectcontour envoy -p '{"spec":{"template":{"spec":{"nodeSelector":{"ingress-ready":"true"},"tolerations":[{"key":"node-role.kubernetes.io/control-plane","operator":"Equal","effect":"NoSchedule"},{"key":"node-role.kubernetes.io/master","operator":"Equal","effect":"NoSchedule"}]}}}}'
```

### - Kong Ingress 생성

- Kong 설치

```
kubectl apply -f https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/single/all-in-one-dbless.yaml
```

- Kong 설정 업데이트

```
kubectl patch deployment -n kong ingress-kong -p '{"spec":{"template":{"spec":{"containers":[{"name":"proxy","ports":[{"containerPort":8000,"hostPort":80,"name":"proxy","protocol":"TCP"},{"containerPort":8443,"hostPort":443,"name":"proxy-ssl","protocol":"TCP"}]}],"nodeSelector":{"ingress-ready":"true"},"tolerations":[{"key":"node-role.kubernetes.io/control-plane","operator":"Equal","effect":"NoSchedule"},{"key":"node-role.kubernetes.io/master","operator":"Equal","effect":"NoSchedule"}]}}}}'
```

### - Nginx Ingress 생성

- Nginx 설치

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

- Nginx 설정 업데이트 

```
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### - Ingress 사용 예제

```
kind: Pod
apiVersion: v1
metadata:
  name: foo-app
  labels:
    app: foo
spec:
  containers:
  - name: foo-app
    image: hashicorp/http-echo:0.2.3
    args:
    - "-text=foo"
---
kind: Service
apiVersion: v1
metadata:
  name: foo-service
spec:
  selector:
    app: foo
  ports:
  # Default port used by the image
  - port: 5678
---
kind: Pod
apiVersion: v1
metadata:
  name: bar-app
  labels:
    app: bar
spec:
  containers:
  - name: bar-app
    image: hashicorp/http-echo:0.2.3
    args:
    - "-text=bar"
---
kind: Service
apiVersion: v1
metadata:
  name: bar-service
spec:
  selector:
    app: bar
  ports:
  # Default port used by the image
  - port: 5678
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: foo-service
            port:
              number: 5678
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: bar-service
            port:
              number: 5678
---
```

## 7. Geteway API 기능 설정
### - Gateway API CRD 설치
```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```

### - Contour Gateway Provisioner 설치
```
kubectl apply -f https://projectcontour.io/quickstart/contour-gateway-provisioner.yaml
```

```
kubectl get pod -n projectcontour -w
```

### - 정상 설정 확인 테스트

```
apiVersion: v1
kind: Namespace
metadata:
  name: tcp-lab
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: tcp-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7.2
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: tcp-lab
spec:
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: contour-gwclass
spec:
  controllerName: projectcontour.io/gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tcp-gateway
  namespace: tcp-lab
spec:
  gatewayClassName: contour-gwclass
  listeners:
    - name: tcp-redis
      protocol: TCP
      port: 6379
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: redis-tcproute
  namespace: tcp-lab
spec:
  parentRefs:
    - name: tcp-gateway
      sectionName: tcp-redis
  rules:
    - backendRefs:
        - name: redis-svc
          port: 6379
```

```
kubectl apply -f tcp-contour-lab.yaml
```


```
kubectl get gateway -n tcp-lab
kubectl get svc -n tcp-lab
```

```
redis-cli -h <GATEWAY-ADDRESS> -p 6379 ping
# PONG 나오면 성공
```

## 8. Storage 기능 설정
