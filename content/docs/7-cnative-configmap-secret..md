---
title: "⚙️ 7. ConfigMap 과 Secret"
weight: 7
date: 2025-03-18
draft: false
---

{{< embed-pdf url="/pdfs/config-secret.pdf" >}}

<br><br>
## 1. ConfigMap
### - 도커에서 매개변수 전달
- 디렉토리 생성 및 소스 코드 작성
docker/fortuneloop.sh
```bash
#!/bin/bash
trap "exit" SIGINT
INTERVAL=$1 #파라메터로 첫번째 매개 변수를 INTERVAL 로 저장
echo "Configured to generate neew fortune every " $INTERVAL " seconds"
mkdir /var/htdocs
while :
do
    echo $(date) Writing fortune to /var/htdocs/index.html
    /usr/games/fortune  > /var/htdocs/index.html
    sleep $INTERVAL
done
```

```bash
chmod 755 fortuneloop.sh
```

- 컨테이너 이미지 생성
docker/Dockerfile
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
### - ConfigMap을 통한 설정
### - ConfigMap 볼륨 사용 (디렉토리)
### - ConfigMap 볼륨 사용 (파일)
## 2. Secret

)