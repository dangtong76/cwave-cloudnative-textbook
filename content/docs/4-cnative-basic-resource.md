---
title: "📦 04. K8s 기본 리소스"
weight: 4
date: 2025-02-02
draft: false
---

{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/k8s-basic.pdf" >}}
<br><br>
## 1. Pod
{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/pod.pdf" >}}

아래와 같이 goapp.yaml 파일을 만듭니다.
### - Pod 생성 해보기

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
kubectl create -f ./nginx.yaml
```
ㄴㄴssss
{{< alert theme='info" >}}
kubectl apply 와 create 차이<br>
- 일반적으로 apply 를 더 많이 씀!!<br>
- create 는 최초 생성시 사용 할 수 있으며, 이미 오브젝트가 있으면 에러남<br>
- apply 는 없으면 만들고, 있으면 업데이트 함
{{< /alert >}}
```bash
kubectl get pod
```
### - POD 서비스 접속 및 로그확인
- 접속을 위한 포트포워딩
```bash
kubectl port-forward nginx-pod 8080:80
```
- Pod의 네트워크 IP 조회
```bash
kubectl get po -o wide
```
- 다른 터미널에서 curl 을 이용해서 테스트
```bash
curl http://<Pod-IP-Address>:8080
```

- POD Bash 접속
```bash
kubectl exec -it nginx-pod -- bash
```
- POD 이벤트 확인하기
```bash
kubectl describe po nginx-pod
```
- 컨테이너 로그 확인 해보기
```bash
kubectl logs nginx-pod
```
### - [연습문제] 4-1. Pod 생성하기
1. 아래 조건을 만족하는 Pod를 생성하는 YAML 파일을 작성하고, 실제로 클러스터에 배포해보세요.
    - Pod 이름: my-redis
    - 컨테이너 이름: redis
    - 컨테이너 이미지: redis:7.2
    - 컨테이너가 6379번 포트로 서비스
{{< answer >}}
apiVersion: v1
kind: Pod
metadata:
  name: my-redis
spec:
  containers:
  - name: redis
    image: redis:7.2
    ports:
    - containerPort: 6379
{{< /answer >}}

2. redis가 실행중인 컨테이너에 bash로 접속후 프로세스를 조회 하세요
    {{< answer >}}
    kubectl exec -it redis-pod -- bash
    {{< /answer >}}
3. 파드의 이벤트 로그를 확인하세요
    {{< answer >}}
    kubectl logs redis-pod
    {{< /answer >}}
4. redis 컨테이너의 로그를 확인하세요
    {{< answer >}}
    kubectl logs redis-pod
    {{< /answer >}}
5. 

## 2. 라벨(Label)과 어노테니션(annotaion)
{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/Label_Annotation.pdf" >}}
<br><br>
### - 라벨(label)

1. goapp-with-label.yaml 이라 파일에 아래 내용을 추가 하여 작성 합니다.
```yml
apiVersion: v1
kind: Pod
metadata:
  name: goapp-pod
  labels:
    env: prod
    tier: backend
spec:
  containers:
  - image: dangtong/goapp
    name: goapp-container
    ports:
    - containerPort: 8080
      protocol: TCP
```
2. yaml 파일을 이용해 pod 를 생성 합니다.
```bash
kubectl apply -f ./goapp-with-label.yaml
```

3. 생성된 POD를 조회 합니다.
```bash
kubectl get po --show-labels
```

4. label 태그를 출력 화면에 컬럼을 분리해서 출력
```bash
kubectl get po -L env -L tier
```

5. Label 추가
```bash
kubectl label po goapp-pod app=web
```
6. label 수정
```bash
kubectl label po goapp-pod app=api --overwrite
```
6. Label 삭제
```bash
kubectl label po goapp-pod app-
```

### - 어노테이션(Annotation)
1. Annotaion 추가
```bash
kubectl annotate pod goapp-pod maker="dangtong" team="k8s-team"
```
2. Annotation 확인
```bash
kubectl describe po goapp-pod
```
3. Annotation 삭제
```bash
kubectl annotate po goapp-pod maker- team-
```

### - [연습문제] 4-2. Label 및 Annotaion 실습
- bitnami/apache 이미지로 Pod 를 만들고 tier=FronEnd, app=apache 라벨 정보를 포함하세요
{{< answer >}}
apiVersion: v1
kind: Pod
metadata:
  name: apache-pod
  labels:
    tier: FrontEnd
    app: apache
spec:
  containers:
  - name: apache
    image: bitnami/apache:latest
    ports:
    - containerPort: 8080
{{< /answer >}}

- Pod 정보를 출력 할때 라벨을 함께 출력 하세요
{{< answer >}}
kubectl get po --show-labels
kubectl get po -L tier -L app
{{< /answer >}}

- app=apache 라벨틀 가진 Pod 만 조회 하세요
{{< answer >}}
kubectl get po -l app=apache
{{< /answer >}}

- 만들어진 Pod에 env=dev 라는 라벨 정보를 추가 하세요
{{< answer >}}
kubectl label po apache-pod env=dev
{{< /answer >}}

- created_by=kevin 이라는 Annotation을 추가 하세요
{{< answer >}}
kubectl annotate pod apache-pod created_by=kevin
{{< /answer >}}

- 모든 라벨과 어노테이션을 삭제 하세요
{{< answer >}}
# 라벨 삭제
kubectl label po apache-pod tier- app- env-
# 어노테이션 삭제
kubectl annotate po apache-pod created_by-
# 라벨 삭제 확인
kubectl get po --show-labels
# 어노테이션 삭제 확인
kubectl describe po apache-pod
{{< /answer >}}

## 3. Namespace

{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/namespace.pdf" >}}
<br><br>
### - 현재 네임스페이스 조회

```bash
kubectl get namespace
```
```bash
kubens
```
<br><br>
### - 특정 네임스페이스의 리소스 조회
```bash
kubectl get po --namespace kube-system
```
```bsh
kubectl get po -n kube-system
```
<br><br>
### - 네임스페이스 생성
1. YAML 파일 작성 : first-namespace.yaml 이름으로 파일 작성
  ```yml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: first-namespace
  ```
2. YAML 파일을 이용한 네이스페이스 생성
```bash
kubectl create -f first-namespace.yaml
```
3. 생성된 네임스페이스 확인
```bash
kubectl get ns
```
4. 명령어 이용한 네임스페이스 생성
```bash
kubectl create ns second-namespace
```
<br><br>
### - 네임스페이스 이동
기본적으로 쿠버네티스에서 지원 하는 방법
```bash
kubectl config set-context --current --namespace=<namespace-name>
```
kubens 를 설치하면 네임스페이스 목록,현재 네임스페이스 확인 하고, 다른 네임스페이스로 이동 가능 
```bash
kubens
```

```bash
kubens kube-system
```
<br><br>
### - 기타 네임스페이스 관련 
- 모든 네임스페이스의 Pod 조회
```bash
kubectl get po -A
```

- 모든 네임스페이스의 리소스 조회
```bash
kubectl get all -A
```

- 특정 네임스페이스에서의 리소스 조회
```bash
kubectl get po -n kube-system
```

- 네임스페이스 레벨 리소스 조회
```bash
kubectl api-resources --namespaced=true
```

- 네임스페이스가 아닌 리소스 조회
```bash
kubectl api-resources --namespaced=false
```

### - 네임스페이스 삭제
```bash
kubectl delete ns first-namespace second-namespace
```

<br><br>
## 4. Probe
{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/Probe.pdf" >}}
<br><br>
### - Liveness Probe 테스트
컨테이너가 살아있는지 확인하고, 실패 시 컨테이너를 재시작합니다.

#### HTTP Liveness Probe
- Liveness Probe 생성
liveness prove는 Pod에 지정된 주소에 Health Check 를 수행하고 실패할 경우 Pod를 다시 시작 합니다.
이때 중요한 점은 단순히 다시 시작만 하는 것이 아니라, 리포지토리로 부터 이미지를 다시 받아 Pod 를 다시 시작 합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/liveness
    args:
    - /server
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```
[K8s.gcr.io/liveness](http://K8s.gcr.io/liveness) 이미지는 liveness 테스트를 위해 만들어진 이미지 입니다.
자세한 사항은 [gihub 의 liveness](https://github.com/kubernetes/kubernetes/blob/master/test/images/agnhost/liveness/server.go) 을 참고 하세요 
Go 언어로 작성 되었으며, 처음 10초 동안은 정상적인 서비스를 하지만, 10초 후에는 에러를 발생 시킵니다. 코드를 자세히 보면 아래와 같습니다.
```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```

- pod 생성
```
kubectl create -f ./liveness-probe-pod.yaml
```

- pod 확인
```bash
kubectl create -f ./liveness-probe-pod.yaml
```

- pod 이벤트 로그 확인
```bash
kubectl describe pod liveness-http
```
#### Command Liveness Probe
liveness-command-pod.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-command-pod
spec:
  containers:
  - name: liveness-command
    image: busybox:latest
    command: ["/bin/sh", "-c", "touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600"]
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

- pod 생성
```
kubectl create -f ./liveness-command-pod.yaml
```

- pod 확인
```bash
kubectl create -f ./liveness-command-pod.yaml
```

- pod 이벤트 로그 확인
```bash
kubectl describe pod liveness-command-pod
```


### - Readiness Probe
컨테이너가 요청을 받을 준비가 되었는지 확인하고, 실패 시 서비스 엔드포인트에서 제거합니다.

#### 실습 예제 1: HTTP Readiness Probe
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-http-pod
spec:
  containers:
  - name: readiness-http
    image: nginx:latest
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 3
      timeoutSeconds: 2
      failureThreshold: 2
```

#### 실습 예제 2: TCP Readiness Probe
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-tcp-pod
spec:
  containers:
  - name: readiness-tcp
    image: redis:7.2
    ports:
    - containerPort: 6379
    readinessProbe:
      tcpSocket:
        port: 6379
      initialDelaySeconds: 5
      periodSeconds: 10
```

### - Startup Probe
애플리케이션의 초기 시작 시간이 긴 경우 사용하며, 시작 프로브가 성공할 때까지 다른 프로브들은 비활성화됩니다.

#### 실습 예제: Startup Probe with Liveness
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-probe-pod
spec:
  containers:
  - name: startup-app
    image: nginx:latest
    ports:
    - containerPort: 80
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 3
```

### - [연습문제] 4-3. Probe 실습

#### 1. Liveness Probe 실습
아래 조건을 만족하는 Pod를 생성하세요:
- Pod 이름: liveness-demo
- 이미지: nginx:latest
- Liveness probe: HTTP GET /health (포트 80)
- 초기 지연: 10초, 주기: 5초, 타임아웃: 3초, 실패 임계값: 3

{{< answer >}}
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
  - name: liveness-demo
    image: nginx:latest
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
{{< /answer >}}

#### 2. Readiness Probe 실습
아래 조건을 만족하는 Pod를 생성하세요:
- Pod 이름: readiness-demo
- 이미지: redis:7.2
- Readiness probe: TCP 소켓 (포트 6379)
- 초기 지연: 5초, 주기: 10초

{{< answer >}}
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
spec:
  containers:
  - name: readiness-demo
    image: redis:7.2
    ports:
    - containerPort: 6379
    readinessProbe:
      tcpSocket:
        port: 6379
      initialDelaySeconds: 5
      periodSeconds: 10
{{< /answer >}}

#### 3. Probe 상태 확인 명령어
```bash
# Pod 상태 확인
kubectl get po

# Pod 상세 정보 확인 (Probe 상태 포함)
kubectl describe po <pod-name>

# Pod 로그 확인
kubectl logs <pod-name>

# Pod 이벤트 확인
kubectl get events --sort-by='.lastTimestamp'
```

#### 4. Probe 테스트 방법
```bash
# 1. Pod 생성
kubectl apply -f liveness-demo.yaml

# 2. Pod 상태 모니터링
kubectl get po -w

# 3. Pod 상세 정보 확인
kubectl describe po liveness-demo

# 4. Pod 로그 확인
kubectl logs liveness-demo

# 5. Pod 삭제
kubectl delete po liveness-demo
```

### - Probe 설정 옵션 설명

#### 공통 옵션:
- `initialDelaySeconds`: 컨테이너 시작 후 첫 번째 프로브 실행까지 대기 시간
- `periodSeconds`: 프로브 실행 주기
- `timeoutSeconds`: 프로브 타임아웃 시간
- `failureThreshold`: 실패 임계값 (이 횟수만큼 실패하면 실패로 간주)
- `successThreshold`: 성공 임계값 (Readiness/Startup probe에서만 사용)

#### 프로브 타입:
1. **httpGet**: HTTP GET 요청
2. **tcpSocket**: TCP 소켓 연결
3. **exec**: 명령어 실행


## 5. Replication Controller
{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/rc.pdf" >}}
<br><br>
### - Replication Controller 생성
```yml
apiVersion: v1
kind: ReplicationController
metadata:
  name: goapp-rc
spec:
  replicas: 3
  selector:
    app: goapp
  template:
    metadata:
      name: goapp-pod
      labels:
        tier: forntend
        app: goapp
        env: prod
        priority:  high
    spec:
      containers:
      - name: goapp-container
        image: dangtong/goapp
        ports:
        - containerPort: 8080
```

### - 생성 확인
```bash
kubectl get po,rc
```

### - Pod 삭제후 변환 확인
```bash
kubectl delete pod goapp-rc-<random-hash>
```

### - Pod 정보 출력
```bash
kubectl get pod --show-labels
```

### - Pod 라벨 변경 및 현상 확인
```bash
kubectl label pod goapp-rc-szv2r app=goapp-exit --overwrite
```
```bash
kubectl get po
```

### - Pod 이미지 변경
```bash
kubectl edit rc goapp-rc
```
이미지 부분을 찾아서 goapp-v2로 변경합니다.

```yml
template:
  metadata:
    creationTimestamp: null
    labels:
      app: goapp
    name: goapp-pod
  spec:
    containers:
    - image: dangtong/goapp-v2 # 이부분을 변경 합닏다.(기존 : dangtong/goapp)
      imagePullPolicy: Always
      name: goapp-container
```

### - Pod 삭제하고 적용
- Pod 조회
```bash
kubectl get pod
```

'''bash
kubectl delete pod goapp-rc-<random-hash>
```

```bash
kubectl get po -o wide
```

### - Pod 스케일아웃
```bash
kubectl scale rc goapp-rc --replicas=5
```

```bash
kubectl get po 
```

### - Replication Controller 삭제
- Pod 는 그래도 두고 Replication Controller 만삭제
```bash
kubectl delete rc goapp-rc --cascade=orphan
```
- 모두 삭제
```bash
kubectl delete rc goapp-rc
```


## 6. Replica Set
{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/rs.pdf" >}}
<br><br>

### - ReplicaSet 생성
Selector 를 작성 할때 **ForntEnd** 이고 **운영계** 이면서 중요도가 **High** 인 POD 에 대해 RS 를 생성 합니다.
```yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: env, operator: In, values: [prod]}
      - {key: priority, operator: NotIn, values: [low]}
  template:
    metadata:
      labels:
        tier: frontend
        env: prod
        priority: high
    spec:
      containers:
      - name: nginx-frontend
        image: nginx:1.29
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f ./rs.yml
```

### - RS 확인
```bash
kubectl get po -o wide
```
```bash
kubectl get rs -o wide
```

```bash
kubectl get po --show-labels
```

### - [연습문제] 4-4. RC,RS 생성 실습
1. Nginx:1.14.2 Pod 3개로 구성된 Replication Controller를 작성하세요
{{< answer >}}
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
{{< /answer >}}

2. Replication Controller만 삭제하세요 (Pod는 유지)
{{< answer >}}
kubectl delete rc nginx-rc --cascade=orphan
{{< /answer >}}

3. 남겨진 Nginx Pod를 관리하는 ReplicaSet을 작성하되 replica 4개로 구성하세요
{{< answer >}}
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 4
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
{{< /answer >}}

4. Nginx Pod를 6개로 Scale Out 하세요
{{< answer >}}
kubectl scale rs nginx-rs --replicas=6
{{< /answer >}}

5. 전체 과정 확인 명령어
{{< answer >}}
# 1. RC 생성
kubectl apply -f nginx-rc.yaml

# 2. RC 상태 확인
kubectl get rc
kubectl get pods -l app=nginx

# 3. RC만 삭제 (Pod 유지)
kubectl delete rc nginx-rc --cascade=orphan

# 4. 남은 Pod 확인
kubectl get pods -l app=nginx

# 5. RS 생성
kubectl apply -f nginx-rs.yaml

# 6. RS 상태 확인
kubectl get rs
kubectl get pods -l app=nginx

# 7. Scale Out
kubectl scale rs nginx-rs --replicas=6

# 8. 최종 확인
kubectl get rs
kubectl get pods -l app=nginx
{{< /answer >}}

## 7. DeamonSet
### - DaemonSet 생성
```yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: goapp-on-ssd
spec:
  selector:
    matchLabels:
      app: goapp-pod
  template:
    metadata:
      labels:
        app: goapp-pod
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: goapp-container
        image: dangtong/goapp
```

```bash
kubectl apply -f ./ds.yaml
```

```bash
kubectl get po,ds
```
조회 하면 Pod 도 존재하지 않고 데몬셋 정보를 조회 해도 모두 0 으로 나옵닏다. 노드에 disk=ssd 라벨이 없기 때문입니다.
따라서, 조건을 만족 시키기 위해 Label 을 노드에 추가 합니다.
### - 노드라벨 추가
- 현재 노드 목록 조회
```bash
kubectl get no 
```
- 특정 노드에 Label 추가
```bash
kubectl label node <node-name> disk=ssd
```
```bash
kubectl get ds -o wide
```

## 8. Deployment
{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/deployment.pdf" >}}
<br><br>

Deployment는 Pod와 ReplicaSet에 대한 선언적 업데이트를 제공하며, 애플리케이션의 배포와 업데이트를 관리합니다.


### - 기본 Deployment 생성

#### 1. nginx Deployment 생성
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
        image: nginx:1.29
        ports:
        - containerPort: 80
```

#### 2. Deployment 생성 및 확인
```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployment
kubectl get pods -l app=nginx
```

### - Deployment 업데이트

#### 1. Change Cause를 포함한 이미지 업데이트
```bash
# kubectl set image 명령어 사용 (change cause 포함)
kubectl set image deployment/nginx-deployment nginx=nginx:1.14.2 --record

# 또는 YAML 파일 수정 후 적용 (change cause 포함)
kubectl apply -f nginx-deployment-v2.yaml --record
```

**참고**: `--record` 플래그는 최신 Kubernetes 버전에서 deprecated되었습니다. Change cause를 추적하려면 `kubectl annotate`를 사용하세요:
```bash
# Annotate를 사용한 change cause 설정 (권장 방법)
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="Update to nginx 1.30 for performance improvements"
```

#### 2. 업데이트 과정 확인
```bash
# 업데이트 상태 확인
kubectl rollout status deployment/nginx-deployment

# 업데이트 이력 확인 (change cause 포함)
kubectl rollout history deployment/nginx-deployment

# 상세 이력 확인
kubectl rollout history deployment/nginx-deployment --revision=2

# 상세 정보 확인
kubectl describe deployment nginx-deployment
```

#### 3. Change Cause 확인 예제
```bash
# 이력 확인 (change cause 표시)
kubectl rollout history deployment/nginx-deployment

# 출력 예시:
# REVISION  CHANGE-CAUSE
# 1         kubectl apply --filename=nginx-deployment.yaml --record=true
# 2         kubectl set image deployment/nginx-deployment nginx=nginx:1.30 --record=true
# 3         kubectl set image deployment/nginx-deployment nginx=nginx:1.31 --record=true
```
### - Deployment 롤백

#### 1. 이전 버전으로 롤백
```bash
# 이전 버전으로 롤백
kubectl rollout undo deployment/nginx-deployment

# 특정 버전으로 롤백
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

#### 2. 롤백 확인
```bash
# 롤백 상태 확인
kubectl rollout status deployment/nginx-deployment

# 롤백 이력 확인
kubectl rollout history deployment/nginx-deployment
```

### - Deployment 스케일링

#### 1. Replica 수 변경
```bash
# 스케일 아웃
kubectl scale deployment nginx-deployment --replicas=5

# 스케일 인
kubectl scale deployment nginx-deployment --replicas=2
```

#### 2. 스케일링 확인
```bash
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx
```

### - Deployment 전략

#### 1. Rolling Update (기본)
- Deployment 생성
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
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
        image: nginx:1.29
        ports:
        - containerPort: 80
```
- 이미지 업데이트
```
kubectl set image deployment/nginx-deployment nginx=nginx:1.14.2 --record
```

- 롤링 업데이트 상태 확인
```bash
kubectl rollout status deployment/nginx-deployment
```
```bash
kubectl get po -w
```
```bash
kubectl get po
```

#### 2. Recreate 전략
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-recreate
spec:
  replicas: 3
  strategy:
    type: Recreate
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
        image: nginx:1.29
        ports:
        - containerPort: 80
```

### - [연습문제] 4-5. Deployment 실습

#### 1. 기본 Deployment 생성
아래 조건을 만족하는 Deployment를 생성하세요:
- Deployment 이름: httpd-deployment
- 이미지: httpd:2.4
- Replica: 3개
- 포트: 80

{{< answer >}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deployment
  labels:
    app: httpd
spec:
  replicas: 3
  selector:
    matchLabels:
      app: httpd
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
      - name: httpd
        image: httpd:2.4
        ports:
        - containerPort: 80
{{< /answer >}}

{{< answer >}}
kubectl annotate deployment/httpd-deployment kubernetes.io/change-cause="initial create"
{{< /answer >}}

#### 2. Deployment 업데이트 (Change Cause 포함)
생성된 Deployment의 이미지를 httpd:latest로 업데이트하고 change cause를 기록하세요.

{{< answer >}}
# 이미지 업데이트
kubectl set image deployment/httpd-deployment httpd=httpd:latest

# Change cause 설정 (권장 방법)
kubectl annotate deployment/httpd-deployment kubernetes.io/change-cause="Update to httpd:latest for latest features"

# 업데이트 상태 확인
kubectl rollout status deployment/httpd-deployment

# Pod 상태 확인
kubectl get pods -l app=httpd

# Change cause 확인
kubectl rollout history deployment/httpd-deployment
{{< /answer >}}

#### 3. Deployment 스케일링
Deployment를 5개 replica로 스케일 아웃하세요.

{{< answer >}}
# 스케일 아웃
kubectl scale deployment httpd-deployment --replicas=5

# 스케일링 확인
kubectl get deployment httpd-deployment
kubectl get pods -l app=httpd
{{< /answer >}}

#### 4. Deployment 롤백 (Change Cause 확인)
- rollout 히스토리를 확안하세요
- version 2 의 상세 상태를 확인하세요
- 이전 버전으로 롤백하고 change cause를 확인하세요.

{{< answer >}}
# 롤백 이력 확인 (change cause 포함)
kubectl rollout history deployment/httpd-deployment

# 특정 버전의 상세 정보 확인
kubectl rollout history deployment/httpd-deployment --revision=2

# 이전 버전으로 롤백
kubectl rollout undo deployment/httpd-deployment
kubectl annotate deployment/httpd-deployment kubernetes.io/change-cause="rollback to 2.4"
# 롤백 상태 확인
kubectl rollout status deployment/httpd-deployment
kubectl get pods -l app=httpd

# 롤백 후 이력 확인
kubectl rollout history deployment/httpd-deployment
{{< /answer >}}

#### 5. Deployment 관리 명령어 (Change Cause 포함)
{{< answer >}}
# Deployment 생성
kubectl apply -f httpd-deployment.yaml

# Change cause 설정 (권장 방법)
kubectl annotate deployment/httpd-deployment kubernetes.io/change-cause="Initial deployment with httpd 2.4"

# Deployment 상태 확인
kubectl get deployment
kubectl describe deployment httpd-deployment

# Pod 확인
kubectl get pods -l app=httpd

# 업데이트 이력 확인 (change cause 포함)
kubectl rollout history deployment/httpd-deployment

# 특정 버전 상세 정보 확인
kubectl rollout history deployment/httpd-deployment --revision=1

# Deployment 삭제
kubectl delete deployment httpd-deployment
{{< /answer >}}

### - Deployment 사용 사례

1. **웹 애플리케이션**: nginx, Apache 등
2. **API 서버**: REST API, GraphQL 등
3. **마이크로서비스**: 각 서비스별 독립적 배포
4. **배치 작업**: 주기적 실행되는 작업

### - 정리
```bash
# 리소스 삭제
kubectl delete deployment nginx-deployment
kubectl delete deployment nginx-deployment-recreate
kubectl delete deployment httpd-deployment
```

## 9. StatefulSet
{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/tatefulset.pdf" >}}
<br><br>
StatefulSet 은 디스크를 사용하기 때문에 스토리지를 학습한 후에 배우도록 할께요!

### - 애플리케이션 이미지 작성
```js
// docker/app.js
const http = require('http');
const os = require('os');
const fs = require('fs');

const dataFile = "/var/data/kubia.txt";
const port  = 8080;

// 파일 존재  유/무 검사
function fileExists(file) {
  try {
    fs.statSync(file);
    return true;
  } catch (e) {
    return false;
  }
}

var handler = function(request, response) {
//  POST 요청일 경우 BODY에 있는 내용을 파일에 기록 함
  if (request.method == 'POST') {
    var file = fs.createWriteStream(dataFile);
    file.on('open', function (fd) {
      request.pipe(file);
      console.log("New data has been received and stored.");
      response.writeHead(200);
      response.end("Data stored on pod " + os.hostname() + "\n");
    });
// GET 요청일 경우 호스트명과 파일에 기록된 내용을 반환 함
  } else {
    var data = fileExists(dataFile) ? fs.readFileSync(dataFile, 'utf8') : "No data posted yet";
    response.writeHead(200);
    response.write("You've hit " + os.hostname() + "\n");
    response.end("Data stored on this pod: " + data + "\n");
  }
};

var www = http.createServer(handler);
www.listen(port);
```
### - 도커 이미지 만들기

```dockerfile
# docker/Dockerfile
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

```bash

docker build <Docker-ID>/nodejs:sfs . --push
docker buildx build  --platform linux/amd64,linux/arm64  -t <Docker-ID>/nodejs:sfs . --push
docker login -u <Docker-ID>
docker push <Docker-ID>/nodejs:sfs
```


### - StatefulSet 생성
```yml
# kubetemp/sts.yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nodejs-sfs
spec:
  selector:
    matchLabels:
      app: nodejs-sfs
  serviceName: nodejs-sfs
  replicas: 2
  template:
    metadata:
      labels:
        app: nodejs-sfs
    spec:
      containers:
      - name: nodejs-sfs
        image: <Your-Docker-ID>/nodejs:sfs
        ports:
        - name: http
          containerPort: 8080
        volumeMounts:
        - name: data
          mountPath: /var/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 1Mi
      accessModes:
      - ReadWriteOnce
      storageClassName: gp2
```

```bash
kubectl apply -f ./sts.yml
```

```bash
kubectl get po -w
```

### - LoadBalancer 생성
```yml
# kubetemp/lb.yml
apiVersion: v1
kind: Service
metadata:
    name: nodesjs-sfs-lb
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: external
      service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
      service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
    type: LoadBalancer
    sessionAffinity: None
    ports:
    - port: 80
      targetPort: 8080
    selector:
        app: nodejs-sfs
```

```bash
kubectl apply -f ./lb.yml
```
- 서비스 확인
```bash
kubectl get svc
```

- 로드밸런서 활성 상태 확인 (State 항목 확인)
```bash
aws elbv2 describe-load-balancers
```

- 데이타 조회
```bash
curl http://<public-domain>
```
- 데이터 입력
```bash
curl -X POST -d "hi, my name is dangtong-1" <public-domain>
curl -X POST -d "hi, my name is dangtong-2" <public-domain>
curl -X POST -d "hi, my name is dangtong-3" <public-domain>
curl -X POST -d "hi, my name is dangtong-4" <public-domain>
curl -X POST -d "hi, my name is dangtong-5" <public-domain>
```
데이터 입력을 반복하에 두개 노드 모드에 데이터를 모두 저장 합니다. 양쪽 노드에 어떤 데이터가 입력 되었는지 기억 하고 다음 단계로 넘어 갑니다.

---

## 10. StatefulSet 연습문제


#### 연습문제용 네임스페이스 생성
```bash
kubectl create namespace statefulset-lab
kubectl config set-context --current --namespace=statefulset-lab
```

---

### 연습문제 1: 기본 StatefulSet 생성

#### 문제
아래 조건을 만족하는 StatefulSet을 생성하세요:


{{< answer >}}
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
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: gp2
      resources:
        requests:
          storage: 1Gi
{{< /answer >}}

#### 확인 명령어
```bash
# StatefulSet 상태 확인
kubectl get statefulset

# Pod 상태 확인
kubectl get pods

# PVC 확인 (AWS EBS 볼륨 확인)
kubectl get pvc
kubectl describe pvc www-web-0

# EBS 볼륨 확인 (AWS CLI)
aws ec2 describe-volumes --filters "Name=tag:kubernetes.io/cluster/*,Values=owned" --query 'Volumes[*].[VolumeId,Size,State]' --output table

# Pod 이름 패턴 관찰
kubectl get pods -l app=nginx

# StorageClass 확인
kubectl get storageclass
kubectl describe storageclass gp2

# 문제 해결: PVC 바인딩 상태 확인
kubectl get pvc -o wide
kubectl describe pvc www-web-0

# EBS CSI Driver 상태 확인
kubectl get pods -n kube-system | grep ebs-csi
kubectl logs -n kube-system deployment/ebs-csi-controller

# 노드에 EBS CSI Driver 설치 확인
kubectl get pods -n kube-system | grep ebs-csi-node
```

#### 확인 사항
- [ ] StatefulSet이 정상적으로 생성됨
- [ ] Pod 이름이 `web-0`, `web-1`, `web-2` 패턴으로 생성됨
- [ ] 각 Pod마다 고유한 PVC가 생성됨
- [ ] Pod가 순서대로 생성됨 (0번부터 시작)


### 연습문제 2: StatefulSet과 Headless Service 연동

#### 문제
기존 StatefulSet과 연동되는 Headless Service를 생성하고, 개별 Pod에 접근하는 방법을 확인하세요.

**요구사항:**
- Service 이름: nginx
- 포트: 80
- Headless Service (clusterIP: None)
- Selector: app=nginx

{{< answer >}}
```yaml
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
{{< /answer >}}

#### DNS 테스트 명령어
```bash
# Service 배포
kubectl apply -f nginx-headless-service.yaml

# DNS 조회 테스트
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup nginx

# 개별 Pod DNS 조회
kubectl run test-pod-dns --image=busybox --rm -it --restart=Never -- nslookup web-0.nginx.statefulset-lab.svc.cluster.local
kubectl run test-pod-dns --image=busybox --rm -it --restart=Never -- nslookup web-1.nginx.statefulset-lab.svc.cluster.local
kubectl run test-pod-dns --image=busybox --rm -it --restart=Never -- nslookup web-2.nginx.statefulset-lab.svc.cluster.local


```

---

### 연습문제 3: 영구 스토리지와 데이터 지속성

#### 문제
StatefulSet의 영구 스토리지 특성을 확인하고 데이터 지속성을 테스트하세요.

#### 3-1. 데이터 쓰기 및 확인
```bash
# 각 Pod에 고유한 데이터 쓰기
kubectl exec web-0 -- sh -c "echo 'Data from web-0 at $(date)' > /usr/share/nginx/html/index.html"
kubectl exec web-1 -- sh -c "echo 'Data from web-1 at $(date)' > /usr/share/nginx/html/index.html"
kubectl exec web-2 -- sh -c "echo 'Data from web-2 at $(date)' > /usr/share/nginx/html/index.html"

# 각 Pod의 데이터 확인
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
kubectl exec web-2 -- cat /usr/share/nginx/html/index.html
```

#### 3-2. Pod 재시작 후 데이터 지속성 확인
```bash
# web-0 Pod 삭제 (재시작 시뮬레이션)
kubectl delete pod web-0

# Pod 재생성 대기
kubectl wait --for=condition=ready pod web-0

# 데이터 지속성 확인
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
```

#### 3-3. StatefulSet 스케일링 테스트
```bash
# StatefulSet을 5개로 확장
kubectl scale statefulset web --replicas=5

# 새로운 Pod들의 데이터 확인
kubectl exec web-3 -- cat /usr/share/nginx/html/index.html
kubectl exec web-4 -- cat /usr/share/nginx/html/index.html

# 다시 3개로 축소
kubectl scale statefulset web --replicas=3
```


---

### 연습문제 4: 고급 StatefulSet 구성

#### 문제
환경변수, 리소스 제한, 헬스체크를 포함한 고급 StatefulSet을 구성하세요.

**요구사항:**
- StatefulSet 이름: web-advanced
- Service 이름: nginx-advanced
- 환경변수: NGINX_HOST, NGINX_PORT
- 리소스 제한: CPU 500m, Memory 128Mi
- Liveness Probe: HTTP GET /, 30초 초기지연, 10초 주기
- Readiness Probe: HTTP GET /, 5초 초기지연, 5초 주기
- PVC: www (1Gi), config (100Mi) - StorageClass: gp2

{{< answer >}}
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-advanced
spec:
  serviceName: "nginx-advanced"
  replicas: 3
  selector:
    matchLabels:
      app: nginx-advanced
  template:
    metadata:
      labels:
        app: nginx-advanced
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: web
        env:
        - name: NGINX_HOST
          value: "statefulset.example.com"
        - name: NGINX_PORT
          value: "80"
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
        - name: config
          mountPath: /etc/nginx/conf.d
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: gp2
      resources:
        requests:
          storage: 1Gi
  - metadata:
      name: config
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: gp2
      resources:
        requests:
          storage: 100Mi
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-advanced
  labels:
    app: nginx-advanced
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx-advanced
```
{{< /answer >}}

#### 배포 및 테스트
```bash
# 기존 StatefulSet 삭제
kubectl delete statefulset web
kubectl delete service nginx

# 새로운 StatefulSet 배포
kubectl apply -f advanced-statefulset.yaml
kubectl apply -f nginx-advanced-service.yaml

# 상태 확인
kubectl get statefulset
kubectl get pods
kubectl get pvc

# EBS 볼륨 확인
kubectl describe pvc www-web-advanced-0
kubectl describe pvc config-web-advanced-0

# AWS EBS 볼륨 상태 확인
aws ec2 describe-volumes --filters "Name=tag:kubernetes.io/cluster/*,Values=owned" --query 'Volumes[*].[VolumeId,Size,State,VolumeType]' --output table

# 헬스체크 테스트
kubectl logs web-advanced-0
kubectl describe pod web-advanced-0
kubectl exec web-advanced-0 -- env | grep NGINX

# 리소스 사용량 확인
kubectl top pods -l app=nginx-advanced
```

#### 확인 사항
- [ ] 고급 StatefulSet이 정상 배포됨
- [ ] 환경변수가 정상 설정됨
- [ ] 리소스 제한이 적용됨
- [ ] 헬스체크가 정상 작동함
- [ ] 여러 PVC가 생성됨

---

### 연습문제 5: StatefulSet 업데이트 전략

#### 문제
StatefulSet의 롤링 업데이트를 수행하고 부분 업데이트를 테스트하세요.

#### 5-1. 롤링 업데이트 수행
```bash
# 이미지를 nginx:1.22로 업데이트
kubectl set image statefulset/web-advanced nginx=nginx:1.22

# 업데이트 진행 상황 확인
kubectl rollout status statefulset/web-advanced

# Pod 상태 확인
kubectl get pods -l app=nginx-advanced
```

#### 5-2. 부분 업데이트 테스트
```bash
# partition을 1로 설정하여 web-0만 업데이트
kubectl patch statefulset web-advanced -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":1}}}}'

# 이미지를 nginx:1.23으로 업데이트
kubectl set image statefulset/web-advanced nginx=nginx:1.23

# 업데이트된 Pod 확인
kubectl get pods -l app=nginx-advanced
```

#### 5-3. 롤백 테스트
```bash
# 이전 버전으로 롤백
kubectl rollout undo statefulset/web-advanced

# 롤백 상태 확인
kubectl rollout status statefulset/web-advanced
```

#### 확인 사항
- [ ] 롤링 업데이트가 순서대로 진행됨
- [ ] 부분 업데이트가 정상 작동함
- [ ] 롤백이 정상적으로 수행됨
- [ ] 업데이트 중에도 서비스가 중단되지 않음

---

### 연습문제 정리

#### 최종 확인 사항
- [ ] StatefulSet의 기본 개념 이해
- [ ] Headless Service와의 연동
- [ ] 영구 스토리지의 동작 방식
- [ ] 고급 구성 요소 적용
- [ ] 업데이트 전략 이해

#### 정리 명령어
```bash
# 모든 리소스 삭제
kubectl delete statefulset --all
kubectl delete service --all
kubectl delete pvc --all

# EBS 볼륨 정리 확인 (삭제되지 않은 볼륨이 있을 수 있음)
aws ec2 describe-volumes --filters "Name=tag:kubernetes.io/cluster/*,Values=owned" --query 'Volumes[*].[VolumeId,Size,State]' --output table

# 네임스페이스 삭제
kubectl delete namespace statefulset-lab

# AWS EBS 볼륨 수동 삭제 (필요시)
# aws ec2 delete-volume --volume-id vol-xxxxxxxxx
```

---

### 혹시나 시간 남으면 추가 도전!!!

#### 도전 1: MySQL StatefulSet 구성
MySQL StatefulSet을 구성하고 데이터 지속성을 확인해보세요.

#### 도전 2: Redis Cluster 구성
Redis Cluster를 StatefulSet으로 구성해보세요.

#### 도전 3: 모니터링 추가
Prometheus와 Grafana를 사용하여 StatefulSet 모니터링을 구성해보세요.

---

### 🔗 참고 자료

- [Kubernetes StatefulSet 공식 문서](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [StatefulSet 기본 튜토리얼](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)
- [StatefulSet 고급 튜토리얼](https://kubernetes.io/docs/tutorials/stateful-application/statefulset-scale/)

---

