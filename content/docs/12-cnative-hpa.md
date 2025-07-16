---
title: "🏹 12. 리소스 제어와 HPA"
weight: 12
date: 2025-03-18
draft: false
---

## 11. 리소스 제어
{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/hpa.pdf" >}}
### 11.1 부하 발생용 애플리 케이션 작성

#### 11.1.1 PHP 애플리 케이션 작성

파일명 : index.php

```php
<?php
  $x = 0.0001;
  for ($i = 0; $i <= 1000000; $i++) {
    $x += sqrt($x);
  }
  echo "OK!";
?>
```

#### 11.1.2 도커 이미지 빌드

```dockerfile
FROM php:5-apache
ADD index.php /var/www/html/index.php
RUN chmod a+rx index.php
```

```bash
docker build -t dangtong/php-apache .
docker login
docker push dangtong/php-apache
```

### 11.2 포드 및 서비스 만들기

#### 12.2.1 Deployment 로 Pod 새성

파일명 : php-apache-deploy.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache-dp
spec:
  selector:
    matchLabels:
      app: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: dangtong/php-apache
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
```

#### 12.2.2 로드밸런서 서비스 작성

파일명 : php-apache-svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: php-apache-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: php-apache
```

```bash
kubectl apply -f ./php-apache-deploy.yaml
kubectl apply -f ./php-apache-svc.yaml
```

### [연습문제] 10-1 Pod 리소스 제어하기

아래 요구 사항에 맞는 deploy 를 구현 하세요

1. Deploy name : nginx
2. image : nginx
3. cpu 200m
4. 메모리 : 300Mi
5. Port : 80

### 12.3 HPA 리소스 생성

#### 12.3.1 HPA 리소스 생성

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=5
```

#### 12.3.2 HPA 리소스 생성 확인

```bash
kubectl get hpa
```

### 12.4 Jmeter 설치 및 구성

#### 12.4.1 Jmeter 설치를 위해 필요한 것들

- JDK 1.8 이상 설치 [오라클 java SE jdk 다운로드](https://www.oracle.com/java/technologies/javase-downloads.html)

- Jmeter 다운로드 [Jmeter 다운로드](https://jmeter.apache.org/download_jmeter.cgi)

  플러그인 다운로드 [Jmeter-plugin 다운로드](https://jmeter-plugins.org/install/Install/)

  - Plugins-manager.jar 다운로드 하여 jmeter 내에 lib/ext 밑에 복사 합니다.  jpgc

#### 12.4.2 Jmeter 를 통한 부하 발생

![jmeter](./img/jmeter.png)

#### 12.4.3 부하 발생후 Pod 모니터링

```bash
$ kubectl get hpa

$ kubectl top pods

NAME                          CPU(cores)   MEMORY(bytes)
nodejs-sfs-0                  0m           7Mi
nodejs-sfs-1                  0m           7Mi
php-apache-6997577bfc-27r95   1m           9Mi


$ kubectl exec -it nodejs-sfs-0 top

Tasks:   2 total,   1 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  4.0 us,  1.0 sy,  0.0 ni, 95.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:   3786676 total,  3217936 used,   568740 free,   109732 buffers
KiB Swap:        0 total,        0 used,        0 free.  2264392 cached Mem
    PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
      1 root      20   0  813604  25872  19256 S  0.0  0.7   0:00.17 node
     11 root      20   0   21924   2408   2084 R  0.0  0.1   0:00.00 top
```

### [연습문제] 12-1 HPA 적용 실습

이미지를 nginx 를 생성하고 HPA 를 적용하세요

- Max : 8
- Min : 2
- CPU 사용량이 40% 가 되면 스케일링 되게 하세요