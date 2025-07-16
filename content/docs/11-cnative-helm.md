---
title: "🏭 11. Helm 차트"
weight: 11
date: 2025-03-18
draft: false
---

## 1. Helm 구성 및 사용

### - Helm  다운로드 및 설치

#### 윈도우 설치

- Chocolatey 설치
PowerShell 을 열어서 아래 명령을 수행 합니다. 이미 google cloud sdk 를 설치 하면서 설치 되었을수 있습니다.

```cmd
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

- Helm 설치

```cmd
choco install kubernetes-helm
```

#### Mac / LINUX 설치

- 수동 설치 방법

```bash
# helm 다운로드
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3

# 실행권한 변경
chmod 700 get_helm.sh

# helm 설치
./get_helm.sh

# 버전 확인
helm version

# Helm Repository 추가
helm repo add stable https://charts.helm.sh/stable

# Repository 업데이트 
helm repo update
```

- Brew 설치

```bash
brew install helm
```

## 2. Mysql Helm 차트 다운로드 및 설치

### - mysql helm 검색

```bash
helm search repo stable/mysql

NAME                CHART VERSION    APP VERSION    DESCRIPTION
stable/mysql        1.6.3            5.7.28         Fast, reliable, scalable, and easy to use open-...
stable/mysqldump    2.6.0            2.4.1          A Helm chart to help backup MySQL databases usi...
```

### - 피키지 메타 정보 보기

```bash
helm show chart stable/mysql

apiVersion: v1
appVersion: 5.7.28
description: Fast, reliable, scalable, and easy to use open-source relational database
  system.
home: https://www.mysql.com/
icon: https://www.mysql.com/common/logos/logo-mysql-170x115.png
keywords:
- mysql
- database
- sql
maintainers:
- email: o.with@sportradar.com
  name: olemarkus
- email: viglesias@google.com
  name: viglesiasce
name: mysql
sources:
- https://github.com/kubernetes/charts
- https://github.com/docker-library/mysql
version: 1.6.3
```

### - mysql helm 차트 설치 및 Deployment

```bash
helm install mysql stable/mysql 

AME: mysql-1588321002
LAST DEPLOYED: Fri May  1 08:16:55 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
mysql-1588321002.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

i99OpY3CRp
To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/mysql-1588321002 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}
```

```bash
helm ls

NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS         C
HART            APP VERSION
mysql-1588321701        default         1               2020-05-01 17:28:25.322363879 +0900 +09 deployed       m
ysql-1.6.3      5.7.28
```

### - helm 차트  uninstall

```bash
heml list

NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS         C
HART            APP VERSION
mysql-1588321701        default         1               2020-05-01 17:28:25.322363879 +0900 +09 deployed       m
ysql-1.6.3      5.7.28


helm uninstall mysql-1588321701
release "mysql-1588321701" uninstalled
```

## 3. Helm 차트 만들기

### - Helm 차트 생성

```bash
helm create nginxstd
```

### - Template 파일 수정

- Charts.yaml 파일 수정

```bash
apiVersion: v2
name: nginx-std
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

- Template/deployment.yaml 파일 생성

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.container.name }}
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: {{ .Values.container.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.container.name }}
        environment: {{ .Values.environment }}
    spec:
      containers:
        - name: {{ .Values.container.name }}
          image: {{ .Values.container.image }}:{{ .Values.container.tag }}
          ports:
            - containerPort: {{ .Values.container.port }}
          env:
            - name: environment
              value: {{ .Values.environment }}
```

- template/service.yaml 파일 생성

```yml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.container.name }}-service
  labels:
    app: {{ .Values.container.name }}
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: {{ .Values.container.port }}
  selector:
    app: {{ .Values.container.name }}
  type: LoadBalancer
```

- values.yaml 파일 생성

```yml
environment: development
container:
  name: nginx
  port: 80
  image: nginx
  tag: latest
replicas: 2
```

### - 테스트 하기

- K8s 오브젝트 생성

```bash
helm install nginxstd ./nginxstd
```

- 삭제

```bash
# 확인
kubectl get all
helm list

# 삭제 
helm uninstall nginxstd
```

## 4. 패키지 및 리포지토리 생성

### - 패키지 생성

```bash
helm package ./nginxstd

mkdir prod

mv ./nginx-std-0.1.0.tgz ./prod/
```

### - helm 리포지토리 파일 생성

```bash
# 리포지토리 파일 생성 (index.yaml)
heml repo index ./prod

# 파일 생성 확인
cat ./prod/index.yaml
```

## 5 Helm 패키지 및 Repository 구성하기

### - Github.com Repository 생성

- repository 생성

![image-20210807010226171](./img/image-20210807010226171.png)

- github page 설정

![image-20210807010420974](./img/image-20210807010420974.png)

### - Git repository 생성 및 동기화

```bash
cd prod

git init

git add .

git brancb -m main 

git commit -a -m "initial commit"

git remote add origin https://github.com/dangtong76/helm-prod.git

git push origin main
```

### - Helm 리포지토리 구성 및 추가

- Git page 로 서비스 되는 Git 리포지토리를 Helm 리포지토리에 추가

```bash
helm repo add helm-prod https://dangtong76.github.io/helm-prod
```

- 추가확인

```bash
helm repo list

helm search repo nginx
```

### - Helm 리포지토리에 redis 추가

- redis 안정버전 차트를 로컬 prod 디렉토리에 다운로드 

```bash
helm search repo redis

helm fetch stable/redis -d ./prod
```

- index.yaml 갱싱

```bash
helm repo index ./prod
```

- git 업데이트

```bash
git status

git add .

git commit -a -m "add redis"

git push origin master
```

- helm update 수행 

```bash
helm repo update

helm search repo redis
```

> 업데이트 없이 "helm search repo redis" 를 검색하면 검색이 되지 않습니다.

## 6 Helm 차트 업그레이드

### - Repository 를 통한 Helm 인스톨

```bash
helm list

helm install nginxstd helm-prod/nginx-std
# 또는 
helm install helm-prod/nginx-std --generate-name

#확인
helm status nginxstd
kubectl get all 
```

### - helm 메니페스트를 통한 차트 변경 및 업데이트

- stage-values.yaml 파일 생성

```bash
environment: development
replicas: 4
```

- helm upgrade 로 차트 변경 적용

```bash
helm upgrade -f ./nginxstd/stage-values.yaml nginxstd helm-prod/nginx-std
```

- helm history 로 확인

```bash
helm history
```

- RollBack 수행

```bash
helm rollback nginxstd 1
```

- Rollback 확인

```bash
helm history nginxstd

helm helm status nginxstd

kubectl get po 
```

### - Helm CLI 옵션을 통한 업그레이드

- 현재 차트의 value 를 화인

```bash
helm show values helm-prod/nginx-std

environment: development
container:
  name: nginx
  port: 80
  image: nginx:1.7.9
  tag: hello
replicas: 2
```

- CLI 옵션을 통한 업그레이드 

```bash
helm upgrade --set replicas=4 --set environment=dev nginxstd helm-prod/nginx-std
```

- 확인 

```bash
helm history

helm status nginxstd

kubectl get po
```

## 7 삭제

```bash
helm uninstall nginxstd
```

---

### [연습문제] 11-1. Helm 차트 설치 실습


**문제:**  
1. Helm 저장소에서 stable/mysql 차트를 검색해보세요.  
2. Helm을 이용해 `mydb`라는 이름으로 MySQL을 설치하고, root 비밀번호를 `rootpw1234`로 지정하세요.  
3. 설치가 완료된 후, kubectl 명령어로 MySQL의 root 비밀번호를 확인해보세요.

**힌트:**  
- `helm search repo` 명령어로 차트를 검색할 수 있습니다.  
- `--set` 옵션으로 차트의 값을 오버라이드할 수 있습니다.  
- 비밀번호는 Secret에 base64로 저장됩니다.

{{<answer>}}
# 1. Helm 차트 검색
```bash
helm search repo stable/mysql
```
# 2. Helm 차트 설치
```bash
helm install mydb stable/mysql --set mysqlRootPassword=rootpw1234
```
# 3. root 비밀번호 확인
```bash
kubectl get secret mydb-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo
```
{{</answer>}}

---

### [연습문제] 11-2. Helm 차트 값 오버라이드 실습

**문제:**  
- Helm을 이용해 `mydb2`라는 이름으로 MySQL을 설치하세요.
- 다음 값을 직접 지정하세요:
  - MySQL 사용자명: `user1`
  - MySQL 사용자 비밀번호: `pw5678`
  - 생성할 데이터베이스 이름: `testdb`
  - root 비밀번호: `rootpw1234`

**힌트:**  
- 여러 값을 오버라이드할 때는 `--set` 옵션을 여러 번 사용하거나, 쉼표(,)로 구분할 수 있습니다.

{{<answer>}}
```bash
helm install mydb2 stable/mysql \
  --set mysqlUser=user1 \
  --set mysqlPassword=pw5678 \
  --set mysqlDatabase=testdb \
  --set mysqlRootPassword=rootpw1234
```
{{</answer>}}

---

### [연습문제] 11-3. Helm 차트 직접 만들기

**문제:**  
1. `nginx-std`라는 이름의 Helm 차트를 생성하세요.
2. 생성된 차트의 `values.yaml` 파일에서 replica 수를 3, 환경(environment)을 prod로 변경하세요.
3. 수정한 차트를 `mynginx`라는 이름으로 설치하세요.

**힌트:**  
- `helm create` 명령어로 차트 뼈대를 만들 수 있습니다.
- `values.yaml` 파일에서 기본값을 원하는 값으로 바꿔주세요.
- 설치 시 차트 디렉토리 경로를 지정해야 합니다.

{{<answer>}}
# 1. 차트 생성
```bash
helm create nginx-std
```
# 2. values.yaml 수정
```yaml
replicaCount: 3
environment: prod
```
# 3. 차트 설치
```bash
helm install mynginx ./nginx-std
```
{{</answer>}}
