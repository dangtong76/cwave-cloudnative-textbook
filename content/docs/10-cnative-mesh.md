---
title: "🕸️ 10. 서비스메시 실습"
weight: 10
date: 2025-03-18
draft: false
---

{{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/istio.pdf" >}}

<br><br>
## 1. Istio 소개
- Istio는 서비스 메시(Service Mesh)로, 마이크로서비스 간의 트래픽 관리, 보안, 모니터링을 제공합니다.
- 본 실습에서는 AWS EKS 환경에서 Istio를 설치하고, Bookinfo 샘플 애플리케이션을 배포해 주요 기능을 체험합니다.

## 2. EKS 클러스터 준비
- AWS CLI, kubectl, eksctl이 설치되어 있어야 합니다.
- EKS 클러스터가 이미 생성되어 있고, kubectl이 해당 클러스터에 연결되어 있어야 합니다.

```
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
```
```bash
# EKS 클러스터 목록 확인
eksctl get cluster

# kubectl 연결 확인
kubectl get nodes
```

<br><br>
## 3. Istioctl 설치
Istio 설치를 위해 istioctl을 다운로드합니다.
```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
```

<br><br>
## 4. Istio 설치
Istio 기본 프로필로 설치합니다.
```bash
istioctl install --set profile=demo -y
```
설치가 완료되면 다음 명령어로 Istio 관련 리소스가 생성되었는지 확인합니다.
```bash
kubectl get pods -n istio-system
```

<br><br>
## 5. 네임스페이스에 Istio 사이드카 자동 주입 활성화
애플리케이션이 배포될 네임스페이스에 label을 추가하여 사이드카가 자동으로 주입되도록 설정합니다.
```bash
kubectl create namespace istio-demo
kubectl label namespace istio-demo istio-injection=enabled
```

<br><br>
## 6. 샘플 애플리케이션(Bookinfo) 배포
Istio 공식 샘플인 Bookinfo 애플리케이션을 배포합니다.
```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n istio-demo
```
애플리케이션이 정상적으로 배포되었는지 확인합니다.
```bash
kubectl get pods -n istio-demo
```

<br><br>
## 7. Istio Gateway 및 VirtualService 설정
Bookinfo 애플리케이션에 외부에서 접근할 수 있도록 Gateway와 VirtualService를 배포합니다.
```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml -n istio-demo
```
Gateway가 정상적으로 생성되었는지 확인합니다.
```bash
kubectl get gateway -n istio-demo
```

<br><br>
## 8. 외부 접속 주소 확인
Istio Ingress Gateway의 EXTERNAL-IP를 확인합니다.
```bash
kubectl get svc istio-ingressgateway -n istio-system
```
EXTERNAL-IP가 할당되면, 아래와 같이 접속 주소를 확인할 수 있습니다.
```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
echo http://$INGRESS_HOST:$INGRESS_PORT/productpage
```
브라우저에서 위 URL로 접속하여 Bookinfo 애플리케이션이 정상적으로 동작하는지 확인합니다.



<br><br>
## 9. Istio 주요 기능 실습
- 트래픽 관리 (VirtualService, DestinationRule)
- 보안 (mTLS, 인증/인가)
- 모니터링 (Kiali, Grafana, Jaeger 등)
```
apt-get install siege

ISTIO_INGRESS_URL=$(kubectl get service/istio-ingress -n istio-ingress -o json | jq -r '.status.loadBalancer.ingress[0].hostname')

siege http://$ISTIO_INGRESS_URL -c 5 -d 10 -t 2M # 5명의 가상 사용자가 , 1~10초 랜덤 딜레이로 , 2분동안 부하 발생
```


각 기능별 실습은 Istio 공식 문서 또는 [aws-samples/istio-on-eks Getting Started](https://github.com/aws-samples/istio-on-eks/blob/main/modules/01-getting-started/README.md)에서 예제를 참고하세요.

<br><br>
## 참고
- [Istio 공식 Getting Started](https://istio.io/latest/docs/setup/getting-started/)
- [aws-samples/istio-on-eks Getting Started](https://github.com/aws-samples/istio-on-eks/blob/main/modules/01-getting-started/README.md)

