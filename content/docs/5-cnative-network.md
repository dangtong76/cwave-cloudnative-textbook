---
title: "🛜 5. 네트워크 서비스"
weight: 5
date: 2025-03-18
draft: false
--- 


## 1. 네티워크 서비스 개념
{{< embed-pdf url="/pdfs/Service.pdf" >}}
<br><br>

## 2. Telepresense 설치
[설치 가이드](https://telepresence.io/docs/install/client) (Install Client, Install Trafiic Manager 참고)
### - Telepresense CLI 로컬 설치
- Linux 설치
```bash
# 1. Download the latest binary (~95 MB):
# AMD
sudo curl -fL https://github.com/telepresenceio/telepresence/releases/latest/download/telepresence-linux-amd64 -o /usr/local/bin/telepresence

# ARM
sudo curl -fL https://github.com/telepresenceio/telepresence/releases/latest/download/telepresence-linux-arm64 -o /usr/local/bin/telepresence

# 2. Make the binary executable:
sudo chmod a+x /usr/local/bin/telepresence
```

- AMD(intel) Mac 설치

```
# 1. Download the binary.
sudo curl -fL https://github.com/telepresenceio/telepresence/releases/latest/download/telepresence-darwin-amd64 -o /usr/local/bin/telepresence

# 2. Make the binary executable:
sudo chmod a+x /usr/local/bin/telepresence
```

- ARM(Apple Silicon) Mac 설치
```
# 1. Ensure that no old binary exists. This is very important because Silicon macs track the executable's signature
# and just updating it in place will not work.
sudo rm -f /usr/local/bin/telepresence

# 2. Download the binary.
sudo curl -fL https://github.com/telepresenceio/telepresence/releases/latest/download/telepresence-darwin-arm64 -o /usr/local/bin/telepresence

# 3. Make the binary executable:
sudo chmod a+x /usr/local/bin/telepresence
```
- Windows 설치

```
# To install Telepresence, run the following commands
# from PowerShell as Administrator.

# 1. Download the latest windows zip containing telepresence.exe and its dependencies (~60 MB):
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest https://github.com/telepresenceio/telepresence/releases/latest/download/telepresence-windows-amd64.zip -OutFile telepresence.zip

# 2. Unzip the telepresence.zip file to the desired directory, then remove the zip file:
Expand-Archive -Path telepresence.zip -DestinationPath telepresenceInstaller/telepresence
Remove-Item 'telepresence.zip'
cd telepresenceInstaller/telepresence

# 3. Run the install-telepresence.ps1 to install telepresence's dependencies. It will install telepresence to
# C:\telepresence by default, but you can specify a custom path by passing in -Path C:\my\custom\path
powershell.exe -ExecutionPolicy bypass -c " . '.\install-telepresence.ps1';"

# 4. Remove the unzipped directory:
cd ../..
Remove-Item telepresenceInstaller -Recurse -Confirm:$false -Force

# 5. Telepresence is now installed and you can use telepresence commands in PowerShell.
```

### - Telepresense Server를 K8s 클러스터에 설치

```
# Telepresence namespace 생성
kubectl create namespace telepresense

# Telepresense 서버를 쿠버네티스에 설치
telepresence helm install -n telepresense

### -  설치 확인
kubectl get all -n telepresence
```

### - 연결 하고 상태 확인하기

```
# Telepresense 프록시 연결
telepresence connect  --manager-namespace telepresense

# 연결 상태 확인
telepresence status
```

## 3. ClusterIP

### - nodes app 생성
- 애플리케이션 작성 
- app.js

```js
const http = require('http');
const os = require('os');

console.log("Kubia server starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("You've hit " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

- Dockerfile 작성
```dockerfile
# FROM 으로 BASE 이미지 로드
FROM node:7

# ADD 명령어로 이미지에 app.js 파일 추가
ADD app.js /app.js

# ENTRYPOINT 명령어로 node 를 실행하고 매개변수로 app.js 를 전달
ENTRYPOINT ["node", "app.js"]
```
- 커테이너 빌드 및 푸시
```bash
docker build -t <DOCKER ID>/nodeapp . --push
```

### - POD 생성
- pod 생성
- nodeapp-deploy.yml
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeapp-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodeapp-pod
  template:
    metadata:
      labels:
        app: nodeapp-pod
    spec:
      containers:
      - name: nodeapp-container
        image: dangtong/nodeapp
        ports:
        - containerPort: 8080
```

### - ClusterIP 서비스 생성
- 서비스 생성
- nodeapp-svc.yml
```yml
apiVersion: v1
kind: Service
metadata:
  name: nodeapp-service
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: nodeapp-pod
```

### - 서비스 확인
- 조회
```bash
kubectl get  po,deploy,svc
```

- Deployment 생성
```bash
kubectl apply -f ./nodeapp-deploy.yml
```

- 서비스 생성
```bash
kubectl apply -f ./nodeapp-svc.yml
```

- port-forward to service
```bash
kubectl port-forward service/nodeapp-service 8080:80
```

- port-forward to pod
```bash
```

### [연습문제] 5-1. ClusterIP
- nginx:1.8.9 이미지를 이용하여 Replica=3 인 Deployment 를 생성하세요
{{< answer >}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8.9
        ports:
        - containerPort: 80
{{< /answer >}}
- nginx 서비스를 로드밸렁싱 하는 서비스를 ClusterIP 로 생성하세요
{{< answer >}}
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
{{< /answer >}}
- kubernetes port-forward를 이용해서 네트워크를 연결하고 curl 명령어로 웹사이트를 조회 하세요
{{< answer >}}
kubectl port-forward service/nginx-svc 8080:80
curl http://localhost:8080
{{< /answer >}}
## 4. NodePort (로컬 쿠버네티스에서 실행)
### - 애플리케이션 생성
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeapp-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodeapp-pod
  template:
    metadata:
      labels:
        app: nodeapp-pod
    spec:
      containers:
      - name: nodeapp-container
        image: dangtong/nodeapp
        ports:
        - containerPort: 8080
```

### - NodePort 생성
```yml
apiVersion: v1
kind: Service
metadata:
  name: node-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
  selector:
    app:  nodeapp-pod
```

```bash
kubectl apply -f ./nodeapp-nodeport.yml
```

- 리소스 조회
```bash
kubectl get po,rs,svc
```

- 노드 조회
```bash
kubectl get no -o wide
```

- 서비스 접속
```bash
curl http://<node-ip>:30123
```


## 5. LoadBalancer
### - Deployment 생성
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeapp-deployment
  labels:
    app: nodeapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodeapp-pod
  template:
    metadata:
      labels:
        app: nodeapp-pod
    spec:
      containers:
      - name: nodeapp-container
        image: dangtong/nodeapp
        ports:
        - containerPort: 8080
```
### - LoadBalancer 생성 (AWS)
- AWS Load Balancer 생성
```yml
apiVersion: v1
kind: Service
metadata:
  name:  nodeapp-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: nodeapp-pod
```
- GCP Loadbalancer 생성
```yml
apiVersion: v1
kind: Service
metadata:
  name:  nodeapp-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: nodeapp-pod
```

### - LoadBalancer 확인 및 접속
EXTERNAL-IP 에 해당하는 URL로 브라우저에서 HTTP로 접속
```bash
kubectl get svc
```

## 6. Ingress

Ingress는 클러스터 외부에서 클러스터 내부 서비스로 HTTP/HTTPS 트래픽을 라우팅하는 규칙을 정의합니다.

### - Ingress Controller 확인
```bash
kubectl get deploy -n kube-system
```

### - 기본 Ingress 예제

#### 1. Deployment 생성
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

#### 2. Service 생성
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

#### 3. Ingress 생성
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb-ingress-class
spec:
  controller: ingress.k8s.aws/alb
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: "alb-ingress-class"
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

### - Ingress 테스트

#### 1. 리소스 생성
```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
kubectl apply -f nginx-ingress.yaml
```

#### 2. Ingress 상태 확인
```bash
aws elbv2 describe-load-balancers
kubectl get ingress
kubectl describe ingress nginx-ingress
```

#### 3. 접속 테스트
```bash
# Ingress IP 확인
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup k8s-default-nginxing-d8df56bf3a-1927562401.ap-northeast-2.elb.amazonaws.com

# 호스트 파일에 도메인 추가 (로컬 테스트용)
echo "<Ingress-Public-IP> nginx.example.com" >> /etc/hosts

# 접속 테스트
curl -H "nginx.example.com" http://nginx.example.com
```

### - 경로 기반 라우팅 Ingress

#### 1. 여러 서비스 생성
```yaml
# app1-deploy.yml
# app1 서비스
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: nginx
        ports:
        - containerPort: 80
---
# app1-svc.yml
apiVersion: v1
kind: Service
metadata:
  name: app1-service
spec:
  selector:
    app: app1
  ports:
  - port: 80
    targetPort: 80
```

```yaml
# app2-deploy.yml
# app2 서비스
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: httpd:2.4
        ports:
        - containerPort: 80
---
# app2-svc.yml
apiVersion: v1
kind: Service
metadata:
  name: app2-service
spec:
  selector:
    app: app2
  ports:
  - port: 80
    targetPort: 80
```

#### 2. 경로 기반 Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb-ingress-class
spec:
  controller: ingress.k8s.aws/alb
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: "alb-ingress-class"
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: apache.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```


#### 3. 접속 테스트

```bash
# Ingress 상태 확인
```bash
aws elbv2 describe-load-balancers
```
# Ingress domain 확인
```bash
kubectl get ingress
```

# Ingress IP 확인
```bash
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup <ingress-domain>
```

# 호스트 파일에 도메인 추가 (로컬 테스트용)
```bash
echo "<Ingress-Public-IP> nginx.example.com apache.example.com" >> /etc/hosts
```

# 접속 테스트
```bash
curl -H "Host: nginx.example.com" http://nginx.example.com/

curl -H "Host: apache.example.com" http://apache.example.com/
```



### - TLS/HTTPS Ingress

#### 1. TLS Secret 생성
```bash
# 자체 서명 인증서 생성
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=vhost.example.com/O=example.com"

# Kubernetes Secret 생성
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

#### 2. TLS Ingress 생성
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - vhost.example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

### - [연습문제] 5-3. Ingress 실습

#### 1. 기본 Ingress 생성
아래 조건을 만족하는 Ingress를 생성하세요:
- Ingress 이름: my-ingress
- 호스트: myapp.local
- 서비스: nginx-service
- 경로: /

{{< answer >}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
{{< /answer >}}

#### 2. 경로 기반 라우팅 Ingress 생성
아래 조건을 만족하는 Ingress를 생성하세요:
- Ingress 이름: path-ingress
- 호스트: example.com
- /api 경로 → api-service
- /web 경로 → web-service

{{< answer >}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
{{< /answer >}}

#### 3. Ingress 테스트
생성된 Ingress를 테스트하세요.

{{< answer >}}
# 1. Ingress 상태 확인
kubectl get ingress
kubectl describe ingress my-ingress

# 2. Ingress IP 확인
kubectl get ingress my-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# 3. 호스트 파일에 도메인 추가
echo "$(kubectl get ingress my-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}') myapp.local" >> /etc/hosts

# 4. 접속 테스트
curl -H "Host: myapp.local" http://$(kubectl get ingress my-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# 5. 포트포워딩으로 테스트
kubectl port-forward service/nginx-service 8080:80 &
curl http://localhost:8080
{{< /answer >}}

### - Ingress Controller 설치 (필요시)

#### Nginx Ingress Controller 설치
```bash
# Helm을 사용한 설치
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx

# 또는 kubectl을 사용한 설치
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```

### - 정리
```bash
# 리소스 삭제
kubectl delete ingress my-ingress
kubectl delete ingress path-ingress
kubectl delete service nginx-service
kubectl delete deployment nginx-deployment
```
## 7. Headless Service

Headless Service는 ClusterIP가 없는 서비스로, DNS 조회 시 모든 Pod의 IP 주소를 직접 반환합니다.

### - Headless Service 생성

- StatefulSet 생성 (Headless Service와 함께 사용)
```yaml
# nginx-sts.yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
```

- Headless Service 생성
```yaml
# headless-svc.yml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```

### - Headless Service 테스트

- 리소스 생성
```bash
kubectl apply -f statefulset.yaml
kubectl apply -f headless-service.yaml
```

- Pod 상태 확인
```bash
kubectl get po,svc
```

- DNS 조회 테스트
```bash
# 임시 Pod 생성하여 DNS 테스트
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx

# 또는 다른 방법
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- sh
# nslookup nginx
# exit
```

- 개별 Pod DNS 조회
```bash
# 특정 Pod의 DNS 이름으로 조회
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup web-0.nginx
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup web-1.nginx
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup web-2.nginx
```

### - 일반 Service vs Headless Service 비교

- 일반 Service 생성 (ClusterIP)
```yaml
# plain-svc.yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### - DNS 조회 결과 비교

- 일반 Service DNS 조회
```bash
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx-clusterip
# 결과: 단일 IP 주소 반환 (Service의 ClusterIP)
```

- Headless Service DNS 조회
```bash
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx-headless
# 결과: 모든 Pod의 IP 주소 반환
```

### - [연습문제] 5-2. Headless Service 실습

#### 1. Redis StatefulSet과 Headless Service 생성
아래 조건을 만족하는 리소스를 생성하세요:
- StatefulSet 이름: redis-sts
- Replica: 3개
- 이미지: redis:7.2
- Headless Service 이름: redis-headless
- 포트: 6379

{{< answer >}}
# redis-sfs-svc.yml
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-sts
spec:
  serviceName: "redis-headless"
  replicas: 3
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
          name: redis
---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
{{< /answer >}}

#### 2. DNS 조회 테스트
Headless Service의 DNS 조회 결과를 확인하세요.

{{< answer >}}
# 리소스 생성
kubectl apply -f redis-statefulset.yaml
kubectl apply -f redis-headless-service.yaml

# DNS 조회 테스트
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup redis-headless

# 개별 Pod DNS 조회
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup redis-sts-0.redis-headless
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup redis-sts-1.redis-headless
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup redis-sts-2.redis-headless
{{< /answer >}}

#### 3. Redis 클라이언트로 연결 테스트
Headless Service를 통해 Redis에 연결해보세요.

{{< answer >}}
# Redis 클라이언트 Pod 생성
kubectl run redis-client --image=redis:7.2 --rm -it --restart=Never -- redis-cli -h redis-sts-0.redis-headless -p 6379 ping

# 또는 포트포워딩을 통한 연결
kubectl port-forward pod/redis-sts-0 6379:6379 &
redis-cli -h localhost -p 6379 ping
{{< /answer >}}

### - Headless Service 사용 사례

1. **StatefulSet과 함께 사용**: 각 Pod가 고유한 ID를 가져야 할 때
2. **서비스 디스커버리**: 클라이언트가 모든 Pod에 직접 연결해야 할 때
3. **데이터베이스 클러스터**: 각 노드가 다른 노드를 직접 알아야 할 때
4. **메시징 시스템**: 각 노드가 다른 노드와 직접 통신해야 할 때

### - 정리
```bash
# 리소스 삭제
kubectl delete statefulset redis-sts
kubectl delete service redis-headless
kubectl delete service nginx
kubectl delete statefulset web
```
