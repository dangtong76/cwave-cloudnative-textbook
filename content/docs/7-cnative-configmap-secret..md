---
title: "⚙️ 7. ConfigMap 과 Secret"
weight: 7
date: 2025-03-18
draft: false
---

{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/config-secret.pdf" >}}

<br><br>
## 1. ConfigMap
### - 도커에서 매개변수 전달
#### . 디렉토리 생성 및 소스 코드 작성
```bash
# docimg/docker/fortuneloop.sh
#!/bin/bash
trap "exit" SIGINT
INTERVAL=$1 #파라메터로 첫번째 매개 변수를 INTERVAL 로 저장
echo "Configured to generate neew fortune every " $INTERVAL " seconds"
mkdir /var/htdocs
while :
do
    echso $(date) Writing fortune to /var/htdocs/index.html
    /usr/games/fortune  > /var/htdocs/index.html
    sleep $INTERVAL
done
```

```bash
chmod 755 fortuneloop.sh
```

#### . 컨테이너 이미지 생성
docimg/docker/Dockerfile
```Dockerfile
FROM ubuntu:latest
RUN apt-get update;  apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
RUN chmod 755 /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"]
CMD ["10"]  # args가 없으면 10초
```

```bash
docker buildx build  --platform linux/amd64,linux/arm64  -t <Your-Docker-ID>/fortune:args . --push
```

- Pod생성
```yml
# kubetemp/config-fortune-indocker-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune5s
spec:
  containers:
  - image: dangtong/fortune:args
    args: ["5"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```

```bash
kubectl apply -f ./config-fortune-indockeer-pod.yaml
```

### - Yml 파일을 통한 매개변수 전달
#### . fortuneloop.sh 작성
```yml
# docimg/yml/fortuneloop.sh
#!/bin/bash
# 매개변수르 받지 않고 환경변로에서 바로 참조
trap "exit" SIGINT
echo "Configured to generate neew fortune every " $INTERVAL " seconds"
mkdir /var/htdocs
while :
do
    echo $(date) Writing fortune to /var/htdocs/index.html
    /usr/games/fortune  > /var/htdocs/index.html
    sleep $INTERVAL
done
```
#### . Dockerfile 작성 및 빌드
```
# docimg/yml/Dockerfile
FROM ubuntu:latest
RUN apt-get update;  apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"]
CMD ["10"]  # args가 없으면 10초
```

```bash
docker buildx build  --platform linux/amd64,linux/arm64  -t <Your-Docker-ID>/fortune:env . --push
```
#### . 매개변수 전달 Pod 작성
```yml
# kubetemp/config-in-yml-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: fortune30s
spec:
  containers:
  - image: dangtong/fortune:env
    env:
    - name: INTERVAL 
      value: "30"
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```

```bash
kubectl apply -f ./config-in-yml-pod.yml
```

### - ConfigMap을 통한 설정
#### . ConfigMap 생성 및 확인
```bash
kubectl create configmap fortune-config --from-literal=sleep-interval=7
```
ConfigMap 생성시 이름은 영문자,숫자, 대시, 밑줄, 점 만 포함 할 수 있습니다.
만약 여러개의 매개변수를 저장 할려면 --from-literal=sleep-interval=5 --from-literal=sleep-count=10 와 같이 from-literal 부분을 여러번 반복해서 입력 하면 됩니다.
```bash
kubectl get cm fortune-config
```


```bash
kubectl describe cm fortune-config
```



```bash
kubectl get configmap fortune-config. -o yaml
```
#### . ConfigMap 환경변수로 전달(특정))
ConfigMap 내의 특정 키만 적용 하려면  configMapKeyRef 를 사용하고 전체를 로드 하려면 configMapRef 를 사용 합니다.

```yml
# kubetemp/config-fortune-mapenv-key-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune7s-key
spec:
  containers:
  - image: dangtong/fortune:env
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```


#### . ConfigMap 환경변수로 전달(전체))
```yml
# kubetemp/config-fortune-mapenv-all-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune7s-all
spec:
  containers:
  - image: dangtong/fortune:env
    envFrom:
    - configMapRef:
        name: fortune-config
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```

```bash
kubectl apply -f ./config-fortune-mapenv-key-pod.yaml
```
```bash
kubectl apply -f ./config-fortune-mapenv-all-pod.yaml
```


접속해서 환경변수를 확인 합니다.
```bash
kubectl exec -it fortune7s-all -- bash
kubectl exec -it fortune7s-key -- bash
```
### - ConfigMap 볼륨 사용 (디렉토리)
#### . 설정파일 작성
```
# kubetemp/config-dir/custom-nginx-config.conf
server {
    listen                8080;
    server_name        www.acron.com;

    gzip on;
    gzip_types text/plain application/xml;
    location / {
        root    /usr/share/nginx/html;
        index    index.html index.htm;
    }
}
```
```bash
kubectl create configmap nginx-config --from-file=config-dir
```
#### . ConfigMap 볼륨 이용한 Pod 생성
```yml
# kubetemp/configmap-volume-dir-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configvol
spec:
  containers:
  - image: nginx:1.7.9
    name: web-server
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    ports:
    - containerPort: 8080
      protocol: TCP
  volumes:
  - name: config
    configMap:
      name: nginx-config
```
서버에 접속해서 디렉토리 구조를 한번 보는것이 좋습니다.
```bash
kubectl apply -f ./configmap-volume-dir-pod.yaml
```
디렉토리 확인
```bash
kubectl exec nginx-configvol -c web-server -- ls /etc/nginx/conf.d
```
파일 내용 확인
```bash
kubectl exec nginx-configvol -c web-server -- cat /etc/nginx/conf.d/custom-nginx-config.conf
```
접속 해서 확인
```bash
kubectl exec -it nginx-pod -- bash
```
#### . Response 확인
```bash
kubectl port-forward  pod/nginx-configvol 8080:8080
# kubectl port-forward  nginx-configvol 8080:8080
```
```bash
curl -H "Accept-Encoding: gzip" -I localhost:8080

- Response - 
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Fri, 17 Apr 2020 06:10:32 GMT
Content-Type: text/html
Last-Modified: Tue, 03 Mar 2020 14:32:47 GMT
Connection: keep-alive
ETag: W/"5e5e6a8f-264"
Content-Encoding: gzip
```

#### . ConfigMap 동적 변경하기
```bash
kubectl edit cm nginx-config
```

**gzip on** 부분을 **gzip off** 로 변경 합니다.
```yml
apiVersion: v1
data:
  nginx-config.conf: "server {\n  listen\t80;\n  server_name\tnginx.acorn.com;\n\n
    \ gzip off;\n  gzip_types text/plain application/xml;\n  location / {\n    root\t/usr/share/nginx/html;\n
    \   index\tindex.html index.htm;\n  }  \n}\n\n\n"
  nginx-ssl-config.conf: '##Nginx SSL Config'
  sleep-interval: |
    25
kind: ConfigMap
metadata:
  creationTimestamp: "2020-04-16T15:58:42Z"
  name: fortune-config
  namespace: default
  resourceVersion: "1115758"
  selfLink: /api/v1/namespaces/default/configmaps/fortune-config
  uid: 182302d8-f30f-4045-9615-36e24b185ecb
```

#### . 변경후 테스트
```bash
curl -H "Accept-Encoding: gzip" -I localhost:8080

- Response - 
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Fri, 17 Apr 2020 06:10:32 GMT
Content-Type: text/html
Last-Modified: Tue, 03 Mar 2020 14:32:47 GMT
Connection: keep-alive
ETag: W/"5e5e6a8f-264"
Content-Encoding: gzip
```

```bash
kubectl  exec nginx-configvol -c web-server -- nginx -s reload
```

```bash
curl -H "Accept-Encoding: gzip" -I localhost:8080

- Response - 
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Fri, 17 Apr 2020 06:10:32 GMT
Content-Type: text/html
Last-Modified: Tue, 03 Mar 2020 14:32:47 GMT
Connection: keep-alive
ETag: W/"5e5e6a8f-264"
```

```bash
kubectl delete all --all
```

### - ConfigMap 볼륨 사용 (파일)

#### . ConfigMap 볼륨 이용한 Pod 생성 
```yml
# kubetemp/configmap-volume-file-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configvol
spec:
  containers:
  - image: nginx:1.7.9
    name: web-server
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/conf.d/default.conf
      subPath: nginx-config.conf 
      readOnly: true
    ports:
    - containerPort: 8080
      protocol: TCP
  volumes:
  - name: config
    configMap:
      name: nginx-config
      defaultMode: 0660
```

```bash
kubectl apply -f ./configmap-volume-file-pod.yaml
```
주의 : configmap 의 key 와 파일명이 일치 해야합니다.
Configmap 의 key 는 파일명 입니다.

```
디렉토리 확인
```bash
kubectl exec nginx-configvol -c web-server -- ls /etc/nginx/conf.d
```
파일 내용 확인
```bash
kubectl exec nginx-configvol -c web-server -- cat /etc/nginx/conf.d/custom-nginx-config.conf
```
접속 해서 확인
```bash
kubectl exec -it nginx-pod -- bash
```

### [연습문제] 7-1. ConfigMap

아래 조건을 만족하는 ConfigMap을 생성하세요.

이름: my-config
데이터:
APP_COLOR=blue
APP_MODE=production
{{<answer>}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  APP_COLOR: blue
  APP_MODE: production
{{</answer>}}

{{<answer>}}
kubectl apply -f my-config.yaml
{{</answer>}}
2. 위 ConfigMap의 모든 key를 환경변수로 사용하는 Pod을 생성하세요.

컨테이너 이름: env-checker
이미지: busybox
명령어: env; sleep 3600
{{<answer>}}
apiVersion: v1
kind: Pod
metadata:
  name: env-checker
spec:
  containers:
  - name: env-checker
    image: busybox
    command: ["sh", "-c", "env; sleep 3600"]
    envFrom:
    - configMapRef:
        name: my-config
{{</answer>}}
{{<answer>}}
kubectl apply -f env-checker.yaml
{{</answer>}}


Pod가 정상적으로 환경변수를 받았는지 확인하세요.
{{<answer>}}
kubectl exec -it env-checker -- env | grep APP_
{{</answer>}}
## 2. Secret