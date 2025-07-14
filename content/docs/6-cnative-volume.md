---
title: "💾 6. 볼륨"
weight: 6
date: 2025-03-18
draft: false
---
{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/volume.pdf" >}}

<br><br>
## 1. EmptyDir
### - Docker 이미지 만들기
- 디렉토리 생성
```bash
$ mkdir -p ./fortune/docimg
$ mkdir -p ./fortune/kubetmp
```
아래와 같이 docker 이미지를 작성하기 위해 bash 로 Application을 작성 합니다.

- fortuneloop.sh 만들기
```bash
#!/bin/bash
trap "exit" SIGINT
mkdir /var/htdocs
while :
do
    echo $(date) Writing fortune to /var/htdocs/index.html
    /usr/games/fortune  > /var/htdocs/index.html
    sleep 10
done
```
- Dockerfile 작성
```dockerfile
# docimg/Dockerfile
FROM ubuntu:latest
RUN apt-get update; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
RUN chmod 755 /bin/fortuneloop.sh
ENTRYPOINT /bin/fortuneloop.sh
```

- Docker 이미지 생성
```bash
docker build -t <DOCKER-ID>/fortune .
docker login -u <DOCKER-ID>
docker push <DOCKER-ID>/fortune
```
- Deployment 작성
html 볼륨을 html-generator 및 web-seerver 컨테이너에 모두 마운트 하였습니다.
html 볼륨에는 /var/htdocs 및 /usr/share/nginx/html 이름 으로 서로 따른 컨테이너에서 바라 보게 됩니다.
다만, web-server 컨테이너는 읽기 전용(reeadOnly) 으로만 접근 하도록 설정 하였습니다.

```yml
# kubetmp/fortune-deploy.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortune-deployment
  labels:
    app: fortune
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fortune
  template:
    metadata:
      labels:
        app: fortune
    spec:
      containers:
      - image: dangtong/fortune
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
kubectl apply -f ./fortune-deploy.yml
```

EmptyDir 을 성능을 위해 메모리에 만들 수 있는데 아래와 같이 설정하면 됩니다.
```yml
  volumes:
    - name: cache-volume
      emptyDir:
        medium: Memory
        sizeLimit: 64Mi
```
- LoadBalancer 생성
```yml
# fortune-lb.yml
apiVersion: v1
kind: Service
metadata:
  name: fortune-lb
spec:
  selector:
    app: fortune
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
  externalIPs:
  - 192.168.0.2
```
```bash
kubectl apply -f ./fortune-lb.yaml
```
- 서비스 확인
```bash
kubectl get svc 
```
서비스 도메인을 확인해서 브라우저로 접속 해봅니다.

## 2. Init Container에 의한 Git 볼룸구성
- Git 계정 생성
GitHub.com 에 자신의 계정으로 로그인하여 'k8s-web' 리포지토리를 만듭니다.

- 소스 파일 작성
```bash
mkdir -p ./gitvolume/html
mkdir -p ./gitvolume/kubetmp
```

```html
<!-- gitvolume/html/index.html -->
<!DOCTYPE html>
<html>
<body>

<h1>K8s Landing Page</h1>

<p>Hello Kubernetes !!!</p>

</body>
</html>
```

- 리포지토리 생성 및 초기화
```bash
# gitvolume/html 디렉토리에서 수행
git init
git add .
git commit -a -m "first commit"
git remote add origin https://github.com/<계정명>/k8s-web.git
git remote -v
git branch -M main
git push origin main
git status
```

- 웹서버 Deployment 작성
```yml
# givolume/kubetmp/gitvolume-deploy.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitvolume-deployment
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
        gitRepo:
          repository: https://github.com/<Your-Repository-ID>/k8s-web.git
          revision: master
          directory: .
```

```bash
kubectl apply -f ./gitvolume-deploy.yaml
```

- LoadBalancer 생성
```yml
# givolume/kubetmp/gitvolume-lb.yml
apiVersion: v1
kind: Service
metadata:
  name: gitvolume-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
```
```bash
kubectl apply -f ./gitvolume-lb.yaml
```
## 3. Persistent DISK 생성
-  AWS EBS 생성
```bash
# AWS
$ aws ec2 create-volume --volume-type gp2 --size 80 --availability-zone ap-northeast-2a

## 삭제
## aws ec2 delete-volume --volume-id vol-038a54dff454064f6

## 조회
## aws ec2 describe-volumes --filters Name=status,Values=available Name=availability-zone,Values=ap-northeast-2a
```
- GCP 클러스터 조회 
```bash
gcloud container clusters list
```
- 디스크 생성
```bash
gcloud compute disks create --size=16GiB --zone asia-northeast1-b  mongodb
```
- 디스크 삭제
```bash
## 삭제
## gcloud compute disks delete mongodb --zone asia-northeast1-b
```
## 4. PV with PVC
### -  PV 생성하기
- AWS 생성하기
```yml
# pv.yml
apiVersion: v1
kind: PersistentVolume
metadata:
   name: mongodb-pv
spec:
  capacity:
    storage: 1Gi
  csi:
    driver: ebs.csi.aws.com
    fsType: ext4
    volumeHandle: vol-xxxxxxxxxxxxx
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.ebs.csi.aws.com/zone
              operator: In
              values:
                - ap-northeast-2a
```
AWS의 경우 ebs csi 드라이버를 설치 했다면 노드에 다음과 같이 라벨이 자동으로 추가 됩니다.
topology.ebs.csi.aws.com/zone=ap-northeast-2b
아래 명령어로 확인해서 ebs csi 드라이버가 설치 되었는지 확인 합니다.
```bash
kubectl get no
```
```bash
kubectl describe no <node-name>
```
- GCP 생성하기
```yml
# pv.yml
apiVersion: v1
kind: PersistentVolume
metadata:
   name: mongodb-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
   pdName: mongodb
   fsType: ext4
```
### - PVC 생성하기
```yml
# pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
```
여기서 중요한 것은 **storageClassName: ""** 을 반드시 명기 해야 한다는 것입니다. 명기 하지 않으면 Default StorageClass 인 gp2 가 자동 할당 됩니다.

```bash
kubectl apply -f ./pvc.yml
```

```bash
kubectl get pvc
```

### - PV,PVC를 이용한 Pod 생성
```yml
# pv-pvc-mongo.yml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
```

```bash
kubectl apply -f ./pv-pvc-mongo.yml
```

```bash
kubectl get po,pv,pvc
```
### - MongoDB 접속 및 데이터 인서트
```bash
kubectl exec -it mongodb -- mongo
```
```
use mystore
```

```
db.foo.insert({"first-name" : "dangtong"})
```

```
db.foo.find()
```
### - MongoDB 재가동 및 데이터 확인
- MongdoDB 중단
```bash
kubectl delete pod mongodb
```

- MongoDB Pod 재생성
```bash
kubectl apply -f ./pv-pvc-mongo.yml
```

- MongoDB 접속 및 확인
```bash
kubectl exec -it mongodb -- mongo
```

```
use mystore
```

```
db.foo.find()
```

- MongoDB 삭제
```bash
kubectl delete po mongodb
```
## 5. SC with PVC
### - SC 확인
클라우드에서 제공하는 Default Storage Class  확인 해보기
```bash
kubectl get sc
```
상세내역 확인
```bash
kubectl describe sc gp2
```

### - SC 생성
- AWS 생성
```yml
# sc.yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
# volumeBindingMode: Immediate
reclaimPolicy: Delete
parameters:
  csi.storage.k8s.io/fstype: ext4
  type: gp2
allowedTopologies:
  - matchLabelExpressions:
    - key: topology.ebs.csi.aws.com/zone
      values:
      - ap-northeast-2a
      - ap-northeast-2b
      - ap-northeast-2c
```
**volumeBindingMode: Immediate** 는 PVC를 생성하는 즉시 EBS가 아무 AZ나 랜덤으로 생성되며, 이후 Pod이 해당 AZ에 없는 노드에 스케줄되면 AZ mismatch 에러 발생할 수 있습니다.  
**WaitForFirstConsumer** 로 설정하면 Pod이 먼저 스케줄된 후, 해당 노드의 AZ를 기준으로 EBS 생성 됩니다.
- GCP 생성
```yml
# sc.yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: pd.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
parameters:
  type: pd-ssd
  replication-type: none
allowedTopologies:
  - matchLabelExpressions:
      - key: topology.gke.io/zone
        values:
          - asia-northeast3-a
          - asia-northeast3-b
```

### - PVC 생성하기
```yml
# pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
   name: mongodb-pvc
spec:
  storageClassName: fast
  resources:
    requests:
      storage: 2Gi
  accessModes:
    - ReadWriteOnce
```

```bash
kubectl apply -f ./pvc.yaml
```

```bash
kubectl get pv,pvc
```

### - PVC,SC 이용한 Pod 생성
```yml
# pvc-sc-mongodb-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
```
