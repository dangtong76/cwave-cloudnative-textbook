---
title: "🤖 09. CICD 고급"
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
gh repo fork https://github.com/dangtong76/istory-web-k8s.git --clone=false
git clone https://github.com/<your-id>/istory-web-k8s.git istory-web
# 플랫폼 리포지토리 Fork with Clone
gh repo fork https://github.com/dangtong76/istory-platform-k8s.git --clone=false
git clone https://github.com/<your-id>/istory-platform-k8s.git istory-platform
```

```
### - 디렉토리 생성하기
labs/istory-platform 에서 수행
```bash
mkdir -p aws-eks/base/istory-app
mkdir -p aws-eks/base/istory-db
mkdir -p aws-eks/base/istory-tools
mkdir -p aws-eks/overlay/aws-dev
mkdir -p aws-eks/overlay/aws-prod
mkdir -p aws-eks/overlay/local-dev
```

### - 파일 생성하기
labs/istory-platform 에서 수행
```bash
touch aws-eks/base/istory-app/istory-app-config.yml
touch aws-eks/base/istory-app/istory-app-deploy.yml
touch aws-eks/base/istory-app/istory-app-lb.yml
touch aws-eks/base/istory-app/kustomization.yml
touch aws-eks/base/istory-db/istory-db-pod.yml
touch aws-eks/base/istory-db/istory-db-lb.yml
touch aws-eks/base/istory-db/istory-db-pvc.yml
touch aws-eks/base/istory-db/istory-db-sc.yml
touch aws-eks/base/istory-db/kustomization.yml
touch aws-eks/base/istory-tools/busybox.yml
touch aws-eks/base/istory-tools/kustomization.yml
touch aws-eks/overlay/aws-dev/kustomization.yml
touch aws-eks/overlay/aws-dev/patch-deploy.yml
touch aws-eks/overlay/aws-dev/patch-lb-annotations.yml
touch aws-eks/overlay/aws-dev/.env.secret
```
### - 개발 네임스페이스 생성
```bash
kubectl create ns istory-dev
```
## 3. 도커 파일 빌드 및 업로드
### - 도커 파일생성
파일위치 : xinfra/docker/Dockerfile
```dockerfile
FROM eclipse-temurin:21-jdk-alpine
VOLUME /tmp
RUN addgroup -S istory && adduser -S istory -G istory
USER istory
WORKDIR /home/istory
COPY *.jar /home/istory/istory.jar
ENTRYPOINT ["java","-jar","/home/istory/istory.jar"]
```

### - 자바 빌드 (Gradle)
gradlew 가 있는 디렉토리 위치에서 실행 합니다.
```bash
./gradlew build -x test
```
빌드 후에는 istory-web-k8s/build/libs/springbootdeveloper-0.0.1-SNAPSHOT.jar 이 생성 됩니다.
이 파일을 도커 build 를 위해서 istory-web-k8s/xinfra/docker 디렉토리 안에 복사 합니다. 하지만 굳이 복사하지 않아도 build.gradle 파일에는 빌드가 성공할 경우 자동 복사하는 구문이 들어 있습니다.
```gradle
task copyJarToBin(type: Copy) {
    from "build/libs/springbootdeveloper-0.0.1-SNAPSHOT.jar"
    into "xinfra/docker/"
}
```
만약 파일이 없다면 아래와 같이 수동으로 복사 하세요!
```
cp build/libs/springbootdeveloper-0.0.1-SNAPSHOT.jar xinfra/docker/
```

### - 도커 파일 빌드 및 업로드
```bash
# xinfra/docker 에서 수행
# 컨테이너 이미지 빌드 
docker build -t <your-docker-hub-id>/istory:1 .
```
```bash
# 멀티 플랫폼 빌드 
docker buildx build  --platform linux/amd64,linux/arm64  -t <your-dockerhub-id>/<image-name> --push .
```
```bash
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
위치 : xinfra/aws-eks/base/istory-app
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
위치 : xinfra/aws-eks/base/istory-app
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
위치 : xinfra/aws-eks/base/istory-app
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
### - kustomization.yml
위치 : xinfra/aws-eks/base/istory-app
```yml
resources:
  - istory-app-config.yml
  - istory-app-deploy.yml
  - istory-app-lb.yml
```

## 5. istory-db base 생성
### - istory-db-lb.yml
위치 : xinfra/aws-eks/base/istory-db/
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
위치 : xinfra/aws-eks/base/istory-db/
```yml
apiVersion: v1
kind: Pod
metadata:
  name: istory-db
  labels:
    app: mysql
spec:
  containers:
    - image: mysql:5.6
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
        - name: initdb
          mountPath: /docker-entrypoint-initdb.d # mysql 컨테이너가 실행되면 최초로 자동 실행되는 디렉토리 
  volumes:
    - name: mysql-persistent-storage
      persistentVolumeClaim:
        claimName: mysql-pv-claim
    - name: initdb
      configMap:
        name: mysql-initdb-config
```
### - istory-db-pvc.yml
위치 : xinfra/aws-eks/base/istory-db
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
위치 : xinfra/aws-eks/base/istory-db
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

### - istory-db-init-config.yml
위치 : xinfra/aws-eks/base/istory-db
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

### kustomization.yml
위치 : xinfra/aws-eks/base/istory-db
```yml
resources:
  - istory-db-pod.yml
  - istory-db-lb.yml
  - istory-db-pvc.yml
  - istory-db-sc.yml
  - istory-db-init-config.yml
```

## 6. istroy-tools base 생성
### - busybox.yml
위치 : xinfra/aws-eks/base/istory-tools
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
위치 : xinfra/aws-eks/base/istory-tools
```yml
resources:
  - busybox.yml
```

## 7. aws-dev overlay 생성
위치 : xinfra/aws-eks/overlay/aws-dev
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
위치 : xinfra/aws-eks/overlay/aws-dev
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
위치 : xinfra/aws-eks/overlay/aws-dev

- kustomiztion.yml 에서 **newTag** 키워드를 사용하면, **kustomize edit** 명령을 통해서 **이미지를 변경** 할 수 있습니다.

- kustomize 에서는 secretGenerator, configMapGenerator 를 제공 합니다. 제너레이터를 사용하면 .env 파일을 통해 정보를 만들면 kustomize가 ConfigMap과 Secret 를 자동으로 만들어 줍니다.
```yml
resources:
  - ../../base/istory-app
  - ../../base/istory-db

namespace: istory-dev

patches:
  - path: patch-lb-annotations.yml
    target:
      kind: Service
      name: istory-app-lb

  - path: patch-deploy.yml
    target:
      kind: Deployment
      name: istory-app-deploy

secretGenerator:
  - name: istory-db-secret
    envs:
      - .env.secret
    # 아래 양식으로 .env.secret 파일을 만드세요
    # MYSQL_USER=myuser
    # MYSQL_PASSWORD=myuserpassword
    # MYSQL_ROOT_PASSWORD=myrootpassword

images:
  # base/istory-app/istory-app-deploy.yml 내의 이미지 이름과 동일해야 변경됨
  - name: <your-docker-hub-account-id>/istory # 변경필요 
    newTag: latest

generatorOptions:
  disableNameSuffixHash: true
```
### - .env.secret 파일 작성
```bash
# 아래 양식으로 .env.secret 파일을 만드세요
MYSQL_USER=myuser
MYSQL_PASSWORD=myuserpassword
MYSQL_ROOT_PASSWORD=myrootpassword
MYSQL_DATABASE=dbname
```
### -  kustomize 설치
- 실행 파일 다운로드
```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
```
- PATH 걸려있는 디렉토리로 이동
```bash
mv kustomize /usr/sbin/
```
- 명령어 실행 해보기
```
kustomize
```

### - kustomize 빌드 해보기
- 빌드가 에러 없이 정상적인 출력을 생성하면 OK!
```bash
kustomize build overlay/aws-dev
```



### - 기타 참고 사항

kustomize 명령어로 이미지를 변경 할때는 아래와 같이 명령어 사용
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



## 8. ArgoCD 서버 설치
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 9. 로드밸런서 추가하기
```bash
# Patch일 경우 Annotation이 없어서 Classic LB가 생성되기 때문에 외부 접속 가능
# 일반적으로 Patch가 아니라 Create일 경우 Network LB 생성되고, 이때는 Annotation 있어야함.
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

## 10. ArgoCD CLI 설치
웹 VSCODE IDE를 사용하는 경우 이미 설치 되어 있음
### - Linux
```bash
# latest Version
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# stable Version
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```
### - Windows
```bash
$version = (Invoke-RestMethod https://api.github.com/repos/argoproj/argo-cd/releases/latest).tag_name
$url = "https://github.com/argoproj/argo-cd/releases/download/" + $version + "/argocd-windows-amd64.exe"
$output = "argocd.exe"

Invoke-WebRequest -Uri $url -OutFile $output
```
### - Mac
```bash
brew install argocd
```

## 11. Argocd 서버 설정
### - Admin 패스워드 알아내기
```bash
argocd admin initial-password -n argocd
```
### - 접속 Endpoint 알아내기
```bash
kubectl get svc argocd-server -n argocd

###### 출력 예시 ######
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)                      AGE
argocd-server   LoadBalancer   172.20.21.170   a7eb417c510ec4551933abc911356e6e-1770617535.ap-northeast-2.elb.amazonaws.com   80:31698/TCP,443:31412/TCP   103s
```
### - Argocd CLI 로그인
```bash
argocd login <EXTERNAL-IP>

###### 출력 예시 ######
WARNING: server certificate had error: tls: failed to verify certificate: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? y
Username: admin
Password: 
'admin:login' logged in successfully
Context 'a7eb417c510ec4551933abc911356e6e-1770617535.ap-northeast-2.elb.amazonaws.com' updated
```

### - 새로운 패스워드로 변경
```bash
argocd account update-password --current-password <현재패스워드> --new-password <새로운패스워드>
```

## 12. 브라우저 기반 설정
### - 리포지토리 설정
- 웹 메뉴 : `Settings` → `Repositories` → `CONNECT REPO` 
| 설정 항목 | 값 | 설명 |
|-----------------------|----------------------|---------------------|
| connection method | VIA HTTPS | Git 리포지토리 접속 방식 |
| Type | git | 리포지토리 타입(git | Helm) |
| Name | istory-dev | 참조 이름 |
| Project | Default | Git 의 브렌치 이름 |
| Repository URL | `https://github.com/<github-id>/istory-platform.git` | Platform 리포지토리 URL |
<br>
- 설정화면 참조
{{< figure src="/cwave-cloudnative-textbook/images/argocd-repo1.png" alt="argocd 리포지토리 설정" class="img-fluid" width="60%" >}}

### - 애플리케이션 설정
- 웹 메뉴 : `Application` → `NEW APP`

| 설정 항목 | 값 | 설명 |
|-----------------------|----------------------|---------------------|
| Application Name | istory-dev | - |
| Project Name | default | - |
| Repository URL | `https://github.com/<github-id>/istory-platform.git` | - |
| Path | overlay | aws-dev |
| Cluster URL | https://kubernetes.default.svc | - |
| Namespace | istory-dev | - |
<br>
- 설정 화면 참조 
{{< figure src="/cwave-cloudnative-textbook/images/argocd-application-1.png" alt="argocd 애플리케이션 설정1" class="img-fluid" width="60%" >}}
{{< figure src="/cwave-cloudnative-textbook/images/argocd-application-2.png" alt="argocd 애플리케이션 설정2" class="img-fluid" width="60%" >}}

### - 데이터베이스 시크릿 생성 하기 (.env.secret 생성하지 않앗을 경우)
데이터베이스 계정 및 패스워드는 **Github Action Secret**을 이용해서 동적으로 생성 하기 때문에 최초에 싱크 시에는 직접 **K8s Secret** 객체를 만들어야 함.
**Workflow** 에서는 **Github Action Secret** 을 참조해서 계속 업데이트(kubectl apply ...) 가능하도록 해야 함.
```bash
kubectl create secret generic istory-db-secret \
--namespace istory-dev \
--from-literal=MYSQL_USER=user \
--from-literal=MYSQL_PASSWORD=user12345 \ 
--from-literal=MYSQL_DATABASE=istory \
--from-literal=MYSQL_ROOT_PASSWORD=admin123
```
### - Sync 하기
- 웹 메뉴 : `Application` → `SYNC` → `Synchronize`
## 13. 명령어 기반 설정 (추후 업데이트 예정)
추후 업데이트 예정 ...


## 14. 워크플로우 환경 설정
### - 자바 소스 수정
Github Action Workflow를 통해 빌드된 애플리케이션을 확인하기 위해 소스에 버전을 추가 합니다.
파일 위치 : `src/main/resources/templates/login.html`

**서비스를 사용하려면 로그인을 해주세요!** 뒷 부분에 배포 버전 추가 **V4.0**
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>로그인</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.6.1/dist/css/bootstrap.min.css">

    <style>
        .gradient-custom {
            background: linear-gradient(to right, rgba(106, 17, 203, 1), rgba(37, 117, 252, 1))
        }
    </style>
</head>
<body class="gradient-custom">
<section class="d-flex vh-100">
    <div class="container-fluid row justify-content-center align-content-center">
        <div class="card bg-dark" style="border-radius: 1rem;">
            <div class="card-body p-5 text-center">
                <h2 class="text-white">LOGIN</h2>
                <p class="text-white-50 mt-2 mb-5">서비스를 사용하려면 로그인을 해주세요! V4.0</p>

                <div class = "mb-2">
                    <form action="/login" method="POST">
                        <input type="hidden" th:name="${_csrf?.parameterName}" th:value="${_csrf?.token}" />
                        <div class="mb-3">
                            <label class="form-label text-white">Email address</label>
                            <input type="email" class="form-control" name="username">
                        </div>
                        <div class="mb-3">
                            <label class="form-label text-white">Password</label>
                            <input type="password" class="form-control" name="password">
                        </div>
                        <button type="submit" class="btn btn-primary">Submit</button>
                    </form>

                    <button type="button" class="btn btn-secondary mt-3" onclick="location.href='/signup'">회원가입</button>
                </div>
            </div>
        </div>
    </div>
</section>
</body>
</html>
```
### - Github 토큰생성
워크플로우 에서 사용할 ACCESS TOKEN 들을 각 사이트에 방문해서 생성 합니다.
- Github Token 생성
Github 사이트에서 아래와 같이 토큰을 생성 합니다.
`Profile` → `settings` → `< > Developer settings` → `🔑 Personal access tokens` → `Fine-grained tokens` → `Generate new token`

- 상세 입력 항목
| 입력 항목 | 입력 값 |
|----------------|------------------------------|
| Token name | personal_access_token |
| Repository access  | **All repositories** 에 체크 |
| Repository Permission | **Read and Write** for Actions, Administration, Codepsaces, Contents, Metadata, Pull requests, Secrets, Variables, Workflows | 

### - Docker Hub 토큰 생성
[도커 허브 사이트](https://hub.docker.com/) 에서 아래와 같이 ACCESS TOKEN을 생성합니다.
`Profile` → `Account Setting` → `🔑 Personal access tokens` → `Generate new token`

- 상세 입력 항목
| 입력 항목 | 입력 값 |
|----------------|------------------------------|
| Access token description | Github Workflow TOKEN |
| Expiration date | 30 days |
| Access permissions | Read & Write | 

### - 워크 플로우 디렉토리 생성
istory-web-k8s 디렉토리에서 수행
```bash
mkdir -p .github/workflows
```
### - create-secret.sh 작성
파일 위치 : .github/workflows/create-secret.sh
```bash
gh api -X PUT repos/<your-github-id>/istory-web-k8s/environments/k8s-dev  --silent
gh secret set DATABASE_URL --env k8s-dev --body "jdbc:mysql:istory-db-lb:3306/istory"
gh secret set MYSQL_DATABASE --env k8s-dev --body "istory"
gh secret set MYSQL_USER --env k8s-dev --body "user"
gh secret set MYSQL_PASSWORD --env k8s-dev --body "user12345"
gh secret set MYSQL_ROOT_PASSWORD --env k8s-dev --body "admin123"

gh secret set DOCKER_USERNAME --env k8s-dev --body "<your-docker-hub-id>"
gh secret set DOCKER_TOKEN --env k8s-dev --body "<your-docker-hub-token>"

gh secret set GIT_ACCESS_TOKEN --env k8s-dev --body "<your-github-access-token>"
gh secret set GIT_USERNAME --env k8s-dev --body "<your-github-id>"
gh secret set GIT_PLATFORM_REPO_NAME --env k8s-dev --body "<your-githhub-repository-name>"

gh secret set K8S_CLUSTER_NAME --env k8s-dev --body "istory"
gh secret set K8S_NAMESPACE --env k8s-dev --body "istory-dev"

gh secret set AWS_ACCESS_KEY_ID --env k8s-dev --body "<your-aws-access-key-id>"
gh secret set AWS_SECRET_ACCESS_KEY --env k8s-dev --body "<your-aws-access-key>"
gh secret set AWS_REGION --env k8s-dev --body "<your-aws-region>"
```
- 파일 권한 부여 및 실행
```bash
chmod +x .github/workflows/create-secret.sh

.github/workflows/create-secret.sh
```

## 15. Workflow 생성
### - MySQL 서비스 및 환경설정
파일 위치 : .github/workflows/istory-aws-eks-dev.yml
```yml
#8
name: Istory Deploy to AWS EKS DEV
on:
  push:
    branches: ['main', 'test']
permissions:
  contents: read
  actions: read
  packages: write
jobs:
  build:
    if: contains(github.event.head_commit.message, '[deploy-dev]') # 특정 태그에만 실행
    runs-on: ubuntu-latest
    environment: k8s-dev # 환경 변수 및 시크릿 저장공간
    env:
      DOCKER_IMAGE: ${{ secrets.DOCKER_USERNAME }}/istory
      DOCKER_TAG: ${{ github.run_number }}
    services:
      mysql:
        image: mysql:8.0
        env:
          # root 계정 비밀번호
          MYSQL_ROOT_PASSWORD: ${{ secrets.MYSQL_ROOT_PASSWORD }} 
          # 사용자 계정
          MYSQL_USER: ${{ secrets.MYSQL_USER }} # user
          # 사용자 계정 비밀번호
          MYSQL_PASSWORD: ${{ secrets.MYSQL_PASSWORD }}
          # 사용자 계정 데이터베이스
          MYSQL_DATABASE: ${{ secrets.MYSQL_DATABASE }} # istory
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3
    steps:
      - name: AWS CLI ActionSet 설정
        uses: aws-actions/configure-aws-credentials@v3
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: kubectl 설치
        run: |
          curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
          chmod +x kubectl
          sudo mv kubectl /usr/local/bin/

      - name: kubeconfig 업데이트
        run: |
          aws eks update-kubeconfig --region ${{ secrets.AWS_REGION }} --name ${{ secrets.K8S_CLUSTER_NAME }}

      - name: kustomize 설치
        run: |
          curl -s https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh | bash
          sudo mv kustomize /usr/local/bin

      - name: JDK 17 설치
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
```

### - 애플리케이션 빌드
파일 위치 : .github/workflows/istory-aws-eks-dev.yml
```yml
      - name: 소스코드 다운로드
        uses: actions/checkout@v4 
        with:
          ref: ${{ github.ref }}
          path: .
  
      - name: 개발용 application.yml 생성
        run: |
          cat > src/main/resources/application.yml << EOF
          spring:
            datasource:
              # url: ${{ secrets.DATABASE_URL }} # 예dbc:mysql://localhost:3306/istory
              url: jdbc:mysql://localhost:3306/istory
              username: ${{ secrets.MYSQL_USER }}
              password: ${{ secrets.MYSQL_PASSWORD }}
              driver-class-name: com.mysql.cj.jdbc.Driver
            jpa:
              database-platform: org.hibernate.dialect.MySQL8Dialect
              hibernate:
                ddl-auto: update
              show-sql: true
            application:
              name: USER-SERVICE
            jwt:
              issuer: user@gmail.com
              secret_key: study-springboot
          management:
            endpoints:
              web:
                exposure:
                  include: health,info
            endpoint:
              health:
                show-details: always
          EOF

      - name: Gradle 설정
        uses: gradle/gradle-build-action@v2
        
      - name: JAVA build with TEST
        run: ./gradlew build

      - name: Docker 디렉토리로 JAR 파일 복사
        run: |
          mkdir -p xinfra/docker/build/libs/
          cp build/libs/*.jar xinfra/docker/
          ls xinfra/docker/
```
### - 컨테이너 빌드 및 업로드
파일 위치 : .github/workflows/istory-aws-eks-dev.yml
```yml
      - name: 컨테이너 이미지 빌드
        run: | 
          docker build ./xinfra/docker -t ${{ secrets.DOCKER_USERNAME }}/istory:${{ env.DOCKER_TAG }} -f ./xinfra/docker/Dockerfile
          docker tag ${{ secrets.DOCKER_USERNAME }}/istory:${{ env.DOCKER_TAG }} ${{ secrets.DOCKER_USERNAME }}/istory:latest

      - name: Docker Hub 로그인
        uses: docker/login-action@v3.0.0
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}
          logout: true

      - name: Docker Hub 이미지 업로드
        run: |
          docker push ${{ secrets.DOCKER_USERNAME }}/istory:${{ env.DOCKER_TAG }}
          docker push ${{ secrets.DOCKER_USERNAME }}/istory:latest
```

- 2.4 이미지 태그 변경
파일 위치 : .github/workflows/istory-aws-eks-dev.yml
```yml
      - name: 서비스 리포지토리 체크아웃
        uses: actions/checkout@v4
        with:
          repository: ${{ secrets.GIT_USERNAME }}/${{ secrets.GIT_PLATFORM_REPO_NAME }}
          ref: ${{ github.ref }} # 바꾸기
          path: .
          token: ${{ secrets.GIT_ACCESS_TOKEN }}
  
      - name: 쿠버네티스 secret 생성 (istory-db-secret)
        run: |
          kubectl create secret generic istory-db-secret \
            --from-literal=MYSQL_USER=${{ secrets.MYSQL_USER }} \
            --from-literal=MYSQL_PASSWORD=${{ secrets.MYSQL_PASSWORD }} \
            --from-literal=DATABASE_URL=${{ secrets.DATABASE_URL }} \
            --from-literal=MYSQL_ROOT_PASSWORD=${{ secrets.MYSQL_ROOT_PASSWORD }} \
            --namespace=${{ secrets.K8S_NAMESPACE}} \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: 이미지 태그 업데이트 (kustomize)
        run: |
          cd overlay/aws-dev
          kustomize edit set image ${{ secrets.DOCKER_USERNAME }}/istory=${{ secrets.DOCKER_USERNAME }}/istory:${{ env.DOCKER_TAG }}
      - name: 서비스 리포지토리 최종 업데이트
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'
          git commit -am "Update image tag to ${{ env.DOCKER_TAG }}"
          git push origin ${{ github.ref_name }} 
```

### - 전체파일
```yml
#8
name: Istory Deploy to AWS EKS DEV
on:
  push:
    branches: [ 'main', 'test' ]
permissions:
  contents: read
  actions: read
  packages: write
jobs:
  build:
    if: contains(github.event.head_commit.message, '[deploy-dev]') # 특정 태그에만 실행
    runs-on: ubuntu-latest
    environment: k8s-dev # 환경 변수 및 시크릿 저장공간
    env:
      DOCKER_IMAGE: ${{ secrets.DOCKER_USERNAME }}/istory
      DOCKER_TAG: ${{ github.run_number }}
    services:
      mysql:
        image: mysql:8.0
        env:
          # root 계정 비밀번호
          MYSQL_ROOT_PASSWORD: ${{ secrets.MYSQL_ROOT_PASSWORD }} 
          # 사용자 계정
          MYSQL_USER: ${{ secrets.MYSQL_USER }} # user
          # 사용자 계정 비밀번호
          MYSQL_PASSWORD: ${{ secrets.MYSQL_PASSWORD }}
          # 사용자 계정 데이터베이스
          MYSQL_DATABASE: ${{ secrets.MYSQL_DATABASE }} # istory
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3
    steps:
      - name: AWS CLI ActionSet 설정
        uses: aws-actions/configure-aws-credentials@v3
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: kubectl 설치
        run: |
          curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
          chmod +x kubectl
          sudo mv kubectl /usr/local/bin/

      - name: kubeconfig 업데이트
        run: |
          aws eks update-kubeconfig --region ${{ secrets.AWS_REGION }} --name ${{ secrets.K8S_CLUSTER_NAME }}

      - name: kustomize 설치
        run: |
          curl -s https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh | bash
          sudo mv kustomize /usr/local/bin

      - name: JDK 17 설치
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: 소스코드 다운로드
        uses: actions/checkout@v4 
        with:
          ref: ${{ github.ref }}
          path: .
  
      - name: 개발용 application.yml 생성
        run: |
          cat > src/main/resources/application.yml << EOF
          spring:
            datasource:
              # url: ${{ secrets.DATABASE_URL }} # 예dbc:mysql://localhost:3306/istory
              url: jdbc:mysql://localhost:3306/istory
              username: ${{ secrets.MYSQL_USER }}
              password: ${{ secrets.MYSQL_PASSWORD }}
              driver-class-name: com.mysql.cj.jdbc.Driver
            jpa:
              database-platform: org.hibernate.dialect.MySQL8Dialect
              hibernate:
                ddl-auto: update
              show-sql: true
            application:
              name: USER-SERVICE
            jwt:
              issuer: user@gmail.com
              secret_key: study-springboot
          management:
            endpoints:
              web:
                exposure:
                  include: health,info
            endpoint:
              health:
                show-details: always
          EOF

      - name: Gradle 설정
        uses: gradle/gradle-build-action@v2
        
      - name: JAVA build with TEST
        run: ./gradlew build -x text

      - name: Docker 디렉토리로 JAR 파일 복사
        run: |
          mkdir -p xinfra/docker/build/libs/
          cp build/libs/*.jar xinfra/docker/
          ls xinfra/docker/
      - name: 컨테이너 이미지 빌드
        run: | 
          docker build ./xinfra/docker -t ${{ secrets.DOCKER_USERNAME }}/istory:${{ env.DOCKER_TAG }} -f ./xinfra/docker/Dockerfile
          docker tag ${{ secrets.DOCKER_USERNAME }}/istory:${{ env.DOCKER_TAG }} ${{ secrets.DOCKER_USERNAME }}/istory:latest

      - name: Docker Hub 로그인
        uses: docker/login-action@v3.0.0
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}
          logout: true

      - name: Docker Hub 이미지 업로드
        run: |
          docker push ${{ secrets.DOCKER_USERNAME }}/istory:${{ env.DOCKER_TAG }}
          docker push ${{ secrets.DOCKER_USERNAME }}/istory:latest

      - name: 서비스 리포지토리 체크아웃
        uses: actions/checkout@v4
        with:
          repository: ${{ secrets.GIT_USERNAME }}/${{ secrets.GIT_PLATFORM_REPO_NAME }}
          ref: ${{ github.ref }} # istory-web-k8s, istory-platform 의 동일한 브렌치 라는 가정
          path: .
          token: ${{ secrets.GIT_ACCESS_TOKEN }}
  
      - name: 쿠버네티스 secret 생성 (istory-db-secret)
        run: |
          kubectl create secret generic istory-db-secret \
            --from-literal=MYSQL_USER=${{ secrets.MYSQL_USER }} \
            --from-literal=MYSQL_PASSWORD=${{ secrets.MYSQL_PASSWORD }} \
            --from-literal=DATABASE_URL=${{ secrets.DATABASE_URL }} \
            --from-literal=MYSQL_ROOT_PASSWORD=${{ secrets.MYSQL_ROOT_PASSWORD }} \
            --namespace=${{ secrets.K8S_NAMESPACE}} \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: 이미지 태그 업데이트 (kustomize)
        run: |
          ls
          cd awsk-eks/overlay/aws-dev
          kustomize edit set image ${{ secrets.DOCKER_USERNAME }}/istory=${{ secrets.DOCKER_USERNAME }}/istory:${{ env.DOCKER_TAG }}

      - name: 서비스 리포지토리 최종 업데이트
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'
          git commit -am "Update image tag to ${{ env.DOCKER_TAG }}"
          git push origin ${{ github.ref_name }} 
```

## 16. ArgoCD 확인 및 웹사이트 접속
애플리케이션 수정한것을 커밋하고 리포지토리에 업로드합니다.
### - Commit 및 Push
```bash
git add .
git commit -am "[aws-dev deploy]add workflow"
git push origin main
```
### - ArgoCD URL 조회 및 접속
URL 접속해서 Commit Hash 값을 비교해서 달라 지는지 확인 후 **Sync** 버튼을 눌러 동기화 합니다.
```bash
kubectl get svc argocd-server -n argocd
```

### - Istory URL 조회 및 접속 
Istory 서비스에 접속해서 로그인 화면의 버전이 맞는지 확인 합니다.
```bash
kubectl get svc -n istory-dev 
```

`http://<site-url>` 로 접속




## [연습문제] 9-1. aws-prod overlay 생성
- istory-prod 라는 별도의 네임스페이스를 만듭니다. 
{{< answer >}}
kubectl create ns istory-prod
{{< /answer >}}
- 데이터 베이스는 AWS RDS를 사용 하도록 테라폼 코드를 추가 합니다.
{{< answer >}}
# RDS Database Subnet Group
resource "aws_db_subnet_group" "rds_subnet_group" {
  name       = "${var.cluster_name}-rds-subnet-group"
  subnet_ids = [for subnet in aws_subnet.private : subnet.id]

  tags = {
    Name = "${var.cluster_name}-rds-subnet-group"
  }
}

# RDS Security Group
resource "aws_security_group" "rds_sg" {
  name        = "${var.cluster_name}-rds-sg"
  description = "Security group for RDS database"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description     = "MySQL from VPC"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.private-sg.id]
  }

  ingress {
    description     = "MySQL from EKS cluster"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [module.eks.cluster_security_group_id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.cluster_name}-rds-sg"
  }
}

# RDS Parameter Group
resource "aws_db_parameter_group" "rds_parameter_group" {
  family = "mysql8.0"
  name   = "${var.cluster_name}-rds-parameter-group"

  parameter {
    name  = "character_set_server"
    value = "utf8mb4"
  }

  parameter {
    name  = "character_set_client"
    value = "utf8mb4"
  }

  parameter {
    name  = "character_set_connection"
    value = "utf8mb4"
  }

  parameter {
    name  = "character_set_database"
    value = "utf8mb4"
  }

  parameter {
    name  = "character_set_results"
    value = "utf8mb4"
  }

  parameter {
    name  = "collation_server"
    value = "utf8mb4_unicode_ci"
  }

  tags = {
    Name = "${var.cluster_name}-rds-parameter-group"
  }
}

# RDS Option Group
resource "aws_db_option_group" "rds_option_group" {
  engine_name              = "mysql"
  major_engine_version     = "8.0"
  name                     = "${var.cluster_name}-rds-option-group"
  option_group_description = "Option group for ${var.cluster_name} RDS"

  tags = {
    Name = "${var.cluster_name}-rds-option-group"
  }
}

# RDS Instance
resource "aws_db_instance" "rds" {
  identifier = "${var.cluster_name}-rds"

  # Engine Configuration
  engine         = "mysql"
  engine_version = var.rds_engine_version
  instance_class = var.rds_instance_class

  # Storage Configuration
  allocated_storage     = var.rds_allocated_storage
  max_allocated_storage = var.rds_max_allocated_storage
  storage_type          = "gp2"
  storage_encrypted     = true

  # Database Configuration
  db_name  = var.rds_database_name
  username = var.rds_username
  password = var.rds_password

  # Network Configuration
  db_subnet_group_name   = aws_db_subnet_group.rds_subnet_group.name
  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  publicly_accessible    = false
  port                   = 3306

  # Backup Configuration
  backup_retention_period = var.rds_backup_retention_period
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  # Performance Configuration
  parameter_group_name = aws_db_parameter_group.rds_parameter_group.name
  option_group_name    = aws_db_option_group.rds_option_group.name

  # Monitoring Configuration
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring_role.arn

  # Deletion Protection
  deletion_protection = false
  skip_final_snapshot = true

  # Tags
  tags = {
    Name        = "${var.cluster_name}-rds"
    Environment = var.environment
  }
}

# IAM Role for RDS Enhanced Monitoring
resource "aws_iam_role" "rds_monitoring_role" {
  name = "${var.cluster_name}-rds-monitoring-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "monitoring.rds.amazonaws.com"
        }
      }
    ]
  })
}

# Attach RDS monitoring policy to the role
resource "aws_iam_role_policy_attachment" "rds_monitoring_policy" {
  role       = aws_iam_role.rds_monitoring_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}

# Outputs
output "rds_endpoint" {
  description = "The connection endpoint for the RDS instance"
  value       = aws_db_instance.rds.endpoint
}

output "rds_port" {
  description = "The port on which the RDS instance accepts connections"
  value       = aws_db_instance.rds.port
}

output "rds_database_name" {
  description = "The name of the database"
  value       = aws_db_instance.rds.db_name
}

output "rds_username" {
  description = "The master username for the database"
  value       = aws_db_instance.rds.username
  sensitive   = true
}

output "rds_identifier" {
  description = "The RDS instance identifier"
  value       = aws_db_instance.rds.identifier
} 
{{< /answer >}}
{{< answer >}}

variable "rds_backup_retention_period" {
  description = "backup retention"
  type        = number
  default     = 10 
}
variable "rds_password" {
  description = "backup retention"
  type        = string
  default     = "user12345"  
}
variable "rds_username" {
  description = "backup retention"
  type        = string
  default     = "user"  
}

variable "rds_database_name" {
  description = "backup retention"
  type        = string
  default     = "istory"  
}

variable "rds_max_allocated_storage" {
  description = "backup retention"
  type        = number
  default     = 100
}

variable "rds_allocated_storage" {
    description = "The allocated storage in gigabytes"
    type        = number
    default     = 20
}

variable "rds_engine_version" {
  description = "The engine version to use"
  type        = string
  default     = "8.0.35"
}

variable "rds_instance_class" {
  description = "The instance type of the RDS instance"
  type        = string
  default     = "db.t3.micro"
}
{{< /answer >}}
- overlay/aws-prod 디렉토리를 kustomzie 에 추가 합니다.
{{< answer >}}
mkdir -p aws-eks/overlay/aws-prod
{{< /answer >}}
- .env.secret 파일 작성합니다.
{{< answer >}}
MYSQL_USER=user
MYSQL_PASSWORD=user12345
MYSQL_ROOT_PASSWORD=admin123
MYSQL_DATABASE=istory
{{< /answer >}}
- kustomiztion.yml 에 DB 부분을 삭제 합니다.(RDS 사용하기 때문에)
{{< answer >}}
resources:
  - ../../base/istory-app

namespace: istory-prod

patches:
  - path: patch-app-config.yml
    target:
      kind: ConfigMap
      name: istory-app-config
      
  - path: patch-lb-annotations.yml
    target:
      kind: Service
      name: istory-app-lb

  - path: patch-deploy.yml
    target:
      kind: Deployment
      name: istory-app-deploy

secretGenerator:
  - name: istory-db-secret
    envs:
      - .env.secret
    # 아래 양식으로 .env.secret 파일을 만드세요
    # MYSQL_USER=myuser
    # MYSQL_PASSWORD=myuserpassword
    # MYSQL_ROOT_PASSWORD=myrootpassword

images:
  # base/istory-app/istory-app-deploy.yml 내의 이미지 이름과 동일해야 변경됨
  - name: <your-docker-hub-account-id>/istory # 변경필요
    newTag: latest

generatorOptions:
  disableNameSuffixHash: true
{{< /answer >}}
- patch-app-config.yml 파이을 만들어서 RDS URL 로 변경 합니다.
{{< answer >}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: istory-app-config
data:
  spring.datasource.url: 'jdbc:mysql://<Your-RDS-End-Poing-URL>:3306/istory'
{{< /answer >}}
- patch-deploy.yml 파일을 만들어서 annotation(istory.io/env: prod) 과 replicas 개수 변경 합니다.
{{< answer >}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istory-app-deploy
  annotations:
    istory.io/env: prod
    istory.io/tier: backend-app
    istory.io/infra: aws
spec:
  replicas: 5
  template:
    spec:
      initContainers:
        - name: check-mysql-ready
          command: ['sh',
                    '-c',
                    'until mysqladmin ping -u ${MYSQL_USER} -p${MYSQL_PASSWORD} -h cwave-rds.cxqoe466wnup.ap-northeast-2.rds.amazonaws.com:3306; do echo waiting for database; sleep 2; done;']
{{< /answer >}}
- patch-lb-annotations.yml 의 annotation(istory.io/env: prod)로 변경
{{< answer >}}
apiVersion: v1
kind: Service
metadata:
  name: istory-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    istory.io/infra: aws
    istory.io/env: prod
    istory.io/tier: app-lb
{{< /answer >}}
- 변경된 코드를 istory-platform-k8s 에 commit 후 push 합니다.
{{< answer >}}
git status
git add .
git commit -am "add yaml"
git push origin main --force
{{< /answer >}}
- ArgoCD 에서 Repository 생성 및 Application 설정 합니다.



## [연습문제] 9-2. ConfigMapGenerator 사용
현재 istory-app-config.yml 파일을 ConfigMapGenerator 를 사용하도록 변경하세요
참조 사이트 : https://env.simplestep.ca/

{{< answer >}}
# .env.config
SPRING_DATASOURCE_URL=jdbc:mysql://istory-db-lb:3306/istory
SPRING_DATASOURCE_DRIVERCLASSNAME=com.mysql.cj.jdbc.Driver
SPRING_JPA_DATABASEPLATFORM=org.hibernate.dialect.MySQLDialect
SPRING_JPA_HIBERNATE_DDLAUTO=update
SPRING_JPA_SHOWSQL=true
SPRING_APPLICATION_NAME=USER-SERVICE
{{< /answer >}}

{{< answer >}}
resources:
  - ../../base/istory-app
  - ../../base/istory-db

namespace: istory-dev

patches:
  - path: patch-lb-annotations.yml
    target:
      kind: Service
      name: istory-app-lb

  - path: patch-deploy.yml
    target:
      kind: Deployment
      name: istory-app-deploy

secretGenerator:
  - name: istory-db-secret
    envs:
      - .env.secret
    # 아래 양식으로 .env.secret 파일을 만드세요
    # MYSQL_USER=myuser
    # MYSQL_PASSWORD=myuserpassword
    # MYSQL_ROOT_PASSWORD=myrootpassword

configMapGenerator:
  - name: istory-app-config
    envs:
      - .env.config
images:
  # base/istory-app/istory-app-deploy.yml 내의 이미지 이름과 동일해야 변경됨
  - name: <your-docker-hub-account-id>/istory # 변경필요
    newTag: latest

generatorOptions:
  disableNameSuffixHash: true
{{< /answer >}}

{{< answer >}}
아래 파일의 모든 내용을 주석 처리 합니다.
xinfra/aws-eks/base/istory-app/istory-app-config.yml 
{{< /answer >}}

{{< answer >}}
kustomize build overlay/aws-dev
{{< /answer >}}
