---
title: "🤖 9. CICD 고급"
weight: 9
date: 2025-03-18
draft: false
---
## 1. 리포지토리 포크 하기
```bash
# 현재 계정 및 연결 상태 확인
gh auth status

# 계정 연결
gh auth login 

# 애플리케이션 리포지토리 Fork with Clone
gh repo fork https://github.com/dangtong76/istory-web-k8s.git 

```
## 2.xinfra 디렉토리 기본구조
### - 디렉토리 구조 생각하기
```bash
xinfra/
├── docker/
│   ├── Dockerfile
│   └── springbootdeveloper-0.0.1-SNAPSHOT.jar
│
└── aws-eks/
    ├── base/
    │   ├── istory-app/
    │   ├── istory-db/
    │   └── istory-tools/
    │
    └── overlay/
        ├── aws-dev/
        ├── aws-prod/
        └── local-dev/
```
### - 디렉토리 생성하기
```bash
mkdir -p xinfra/aws-eks/base/istory-app
mkdir -p xinfra/aws-eks/base/istory-db
mkdir -p xinfra/aws-eks/base/istory-tools
mkdir -p xinfra/aws-eks/overlay/aws-dev
mkdir -p xinfra/aws-eks/overlay/aws-prod
mkdir -p xinfra/aws-eks/overlay/local-dev
```

## 3. 도커 파일 빌드 및 업로드
### - 도커 파일생성
파일위치 : xinfra/docker/Dockerfile
```dockerfile
FROM eclipse-temurin:17-jdk-alpine
VOLUME /tmp
RUN addgroup -S istory && adduser -S istory -G istory
USER istory
WORKDIR /home/istory
COPY *.jar /home/istory/istory.jar
ENTRYPOINT ["java","-jar","/home/istory/istory.jar"]
```
### - 도커 파일 빌드 및 업로드
```bash
# 컨테이너 이미지 빌드
docker build -t <your-docker-hub-id>/istory:1 .

# latest 태그 만들기
docker tag <your-docker-hub-id>/istory:1 <your-docker-hub-id>/istory:latest

# Docker hub 리포지토리 로그인
docker login --username <your-docker-hub-id>

# 업로드
docker push <your-docker-hub-id>/istory:1
docker push <your-docker-hub-id>/istory:latest
```

## 4. istory-app base 생성
### - istory-app-config.yml
위치 : xinfra/istory-platform/base/istory-app
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istory-app-config
data:
  spring.datasource.url: 'jdbc:mysql://istory-db-lb:3306/istory'
  spring.datasource.driver-class-name: 'com.mysql.cj.jdbc.Driver'
  spring.jpa.database-platform: 'org.hibernate.dialect.MySQLDialect'
  spring.jpa.hibernate.ddl-auto: 'update'
  spring.jpa.show-sql: 'true'
  spring.application.name: 'USER-SERVICE'
```
### - istory-app-deploy.yml
위치 : xinfra/istory-platform/base/istory-app
컨테이너 이미지 이름을 반드시 자신의 Docker Hub 계정으로 변경 해야함.

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  istory-app-deploy
  labels:
    app:  istory-app
spec:
  selector:
    matchLabels:
      app: istory-app
  replicas: 3
  template:
    metadata:
      labels:
        app: istory-app
    spec:
      initContainers:
        - name: check-mysql-ready
          image: mysql:8.0
          env:
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: istory-db-secret
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: istory-db-secret
                  key: MYSQL_PASSWORD
          command: ['sh',
                    '-c',
                    'until mysqladmin ping -u ${MYSQL_USER} -p${MYSQL_PASSWORD} -h istory-db-lb; do echo waiting for database; sleep 2; done;']
      containers:
        - name: istory
          image: <your-docker-hub-account-id>/istory:latest # 변경필요
          envFrom:
            - configMapRef:
                name: istory-app-config
          env:
            - name: spring.datasource.password
              valueFrom:
                secretKeyRef:
                  name: istory-db-secret
                  key: MYSQL_PASSWORD
            - name: spring.datasource.username
              valueFrom:
                secretKeyRef:
                  name: istory-db-secret
                  key: MYSQL_USER
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 3
            successThreshold: 2
            failureThreshold: 3
            periodSeconds: 20
          ports:
            - containerPort:  3306
              name: istory
      volumes:
        - name: application-config
          configMap:
            name: istory-app-config
      restartPolicy: Always
```
### - istory-app-lb.yml
위치 : xinfra/istory-platform/base/istory-app
```yml
apiVersion: v1
kind: Service
metadata:
  name: istory-app-lb
spec:
  type: LoadBalancer
  selector:
    app: istory-app
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 800
  ports:
    - name: istory-app-lb
      protocol: TCP
      port: 80
      targetPort: 8080
```
### - kustomiztion.yml
위치 : xinfra/istory-platform/base/istory-app
```yml
resources:
  - istory-app-config.yml
  - istory-app-deploy.yml
  - istory-app-lb.yml
```

## 5. istory-db base 생성
### - istory-db-lb.yml
위치 : xinfra/istory-platform/base/istory-db/
```yml
apiVersion: v1
kind: Service
metadata:
  name: istory-db-lb
spec:
  selector:
    app: mysql
  ports:
    - name: mysql-db-lb
      protocol: TCP
      port: 3306
```
### - istory-db-pod.yml
위치 : xinfra/istory-platform/base/istory-db/
```yml
apiVersion: v1
kind: Pod
metadata:
  name: istory-db
  labels:
    app: mysql
spec:
  containers:
    - image: mysql/mysql-server
      name: mysql
      envFrom:
        - secretRef:
            name: istory-db-secret
      ports:
        - containerPort: 3306
          name: mysql
      volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
  volumes:
    - name: mysql-persistent-storage
      persistentVolumeClaim:
        claimName: mysql-pv-claim
```
### - istory-db-pvc.yml
위치 : xinfra/istory-platform/base/istory-db
```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: mysql
spec:
  storageClassName: gp2-persistent
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```
### - istory-db-sc.yml
위치 : xinfra/istory-platform/base/istory-db
```yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2-persistent
provisioner: kubernetes.io/aws-ebs
reclaimPolicy: Retain
parameters:
  type: gp2
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
```
### kustomiztion.yml
위치 : xinfra/istory-platform/base/istory-db
```yml
resources:
  - istory-db-pod.yml
  - istory-db-lb.yml
  - istory-db-pvc.yml
  - istory-db-sc.yml
```

## 6. istroy-tools base 생성
### - busybox.yml
위치 : xinfra/istory-platform/base/istory-tools
```yml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-pod
spec:
  containers:
    - name: ubuntu-container
      image: ubuntu
      command: ["sleep", "infinity"]
```

### - kustomization.yml
위치 : xinfra/istory-platform/base/istory-tools
```yml
resources:
  - busybox.yml
```

## 7. aws-dev overlay 생성
### - patch-deploy.yml
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istory-app-deploy
  annotations:
    istory.io/env: dev
    istory.io/tier: backend-app
    istory.io/infra: aws
spec:
  replicas: 1
```
### - patch-lb-annotaions.yml
```yml
apiVersion: v1
kind: Service
metadata:
  name: istory-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    istory.io/infra: aws
    istory.io/env: dev
    istory.io/tier: app-lb
```
### - kustomization.yml


- kustomiztion.yml newTag 사용하기
```yml
resources:
  - ../../base/istory-app
  - ../../base/istory-db

namespace: istory-dev

patches:
  - path: patch-lb-annotations.yaml
    target:
      kind: Service
      name: istory-app-lb
  - path: patch-deploy.yaml
    target:
      kind: Deployment
      name: istory-app-deploy

images:
  # base/istory-app/istory-app-deploy.yml 내의 이미지 이름과 동일해야 변경됨
  - name: <your-docker-hub-account-id>/istory # 변경필요 
    newTag: latest

generatorOptions:
  disableNameSuffixHash: true
```
kustomization.yml 에서 변경시에는 아래와 같이 명령어 사용
```bash
# 커밋 해시를 이용한 태깅 방법
kustomize edit set image istory=dagntong76/istory:${{ github.sha }}
# 워크플로우 수행 횟수를 이용한 태깅 방법 
kustomize edit set image istory=dagntong76/istory:${{ github.run_number }}
```
- patch-deploy.yml 에서 사용하기기
컨테이너 이미지 부분을 자신의 Docker Hub 계정의 이미지로 변경
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istory-app-deploy
  annotations:
    istory.io/env: dev
    istory.io/tier: backend-app
    istory.io/infra: aws
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: istory
          image: <your-docker-hub-account-id>/istory:latest # 변경필요
```
이 경우는 아래와 같이 workflow 에서 SED 를 사용

```bash
sed -i "s|image: .*|image: dangtong76/istory:${{ github.sha }}|" overlay/aws-dev/patch-deploy.yml
```


## 연습문제 aws-prod overlay 생성
- istory prod 요구사항
  -  AWS RDS 를 사용합니다.
  -  Replica 개수를 3로 합니다.
  -  istory.io 관련 annotation을 prod 기준으로 수정

