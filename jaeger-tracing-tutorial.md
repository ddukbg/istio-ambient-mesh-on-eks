# k8s 환경에서 Jaeger 설치 및 테스트

## 1. Cert-manager 설치 및 Jaeger Operator Helm 차트 설치

### 1.1 Cert-manager 설치
Jaeger Operator의 인증서 문제를 해결하기 위해 cert-manager를 먼저 설치했습니다.
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```
설치 후 cert-manager 관련 Pod들이 정상적으로 Running 상태인지 확인했습니다.

### 1.2 Helm을 사용한 Jaeger Operator 설치
`observability` 네임스페이스에 Jaeger Operator를 Helm 차트로 설치했습니다.
```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

helm install jaeger-operator jaegertracing/jaeger-operator \
  --namespace observability \
  --create-namespace \
  --set rbac.clusterRole=true
```
인증서 관련 오류가 발생하자, self-signed 인증서를 수동으로 생성하여 문제를 해결했습니다.

### 1.3 Self-signed Issuer 및 Certificate 생성
Jaeger Operator 웹훅에 올바른 인증서를 제공하기 위해 다음을 수행했습니다.

#### a. Self-signed Issuer 생성
```bash
kubectl apply -n observability -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
  namespace: observability
spec:
  selfSigned: {}
EOF
```

#### b. 인증서 재생성 (웹훅 도메인 포함)
```bash
kubectl apply -n observability -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: jaeger-operator-service-cert
  namespace: observability
spec:
  secretName: jaeger-operator-service-cert
  commonName: jaeger-operator.observability.svc
  dnsNames:
  - jaeger-operator.observability.svc
  - jaeger-operator-webhook-service.observability.svc
  issuerRef:
    name: selfsigned-issuer
    kind: Issuer
EOF
```
이로써 Jaeger Operator 웹훅의 인증서 검증 오류를 해결하고, Jaeger CR 생성을 성공시켰습니다.

## 2. Jaeger 인스턴스 생성 및 UI 접근

### 2.1 Jaeger 인스턴스 생성
```bash
kubectl apply -n observability -f - <<EOF
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: simplest
EOF
```
`jaegertracing.io/simplest` 인스턴스가 정상적으로 생성되었습니다.

### 2.2 Jaeger 관련 리소스 상태 확인
```bash
kubectl get pods -n observability
kubectl get svc -n observability
kubectl get ingress -n observability
```
Jaeger UI에 접근하기 위해 포트 포워딩 사용:
```bash
kubectl port-forward -n observability svc/simplest-query 16686:16686
```
브라우저에서 [http://localhost:16686](http://localhost:16686)을 통해 Jaeger UI에 접속하여 확인했습니다.

## 3. OTLP 수신기 활성화 및 HotROD 배포

### 3.1 Jaeger Collector에서 OTLP 수신기 활성화
```bash
kubectl patch jaeger simplest -n observability --type=merge -p '{
  "spec": {
    "collector": {
      "otlp": {
        "enabled": true
      }
    }
  }
}'
```
이를 통해 Jaeger Collector가 포트 4318에서 OTLP 트레이스를 수신할 수 있도록 설정했습니다.

### 3.2 HotROD 애플리케이션 배포
수정된 Deployment YAML(`hot.yaml`)을 사용하여 HotROD 프론트엔드를 배포했습니다.  
YAML 주요 내용:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hotrod-frontend
  namespace: hotrod-demo
  labels:
    app: hotrod
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hotrod
      tier: frontend
  template:
    metadata:
      labels:
        app: hotrod
        tier: frontend
    spec:
      containers:
      - name: hotrod-frontend
        image: jaegertracing/example-hotrod:latest
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://simplest-collector.observability.svc.cluster.local:4318"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hotrod-frontend
  namespace: hotrod-demo
spec:
  type: ClusterIP
  selector:
    app: hotrod
    tier: frontend
  ports:
  - port: 8080
    targetPort: 8080
```
기존 Deployment가 있었다면 삭제한 후 새 YAML을 적용했습니다:
```bash
kubectl delete deployment hotrod-frontend -n hotrod-demo
kubectl apply -f hot.yaml
```
Pods와 서비스가 정상적으로 생성되었는지 확인했습니다.

### 3.3 HotROD 테스트 및 트레이싱 확인
포트 포워딩을 설정하고:
```bash
kubectl port-forward -n hotrod-demo svc/hotrod-frontend 8080:8080
```
브라우저에서 [http://localhost:8080](http://localhost:8080)에 접속하여 트래픽을 발생시켰습니다.

Jaeger UI에서 HotROD 관련 서비스 트레이스가 수집되는지 확인했습니다.

![image](https://github.com/user-attachments/assets/f2afef0c-0cd6-47f2-ae1b-f4c8fff9531b)


## 4. Jaeger 관련 리소스 삭제 및 정리

Istio Ambient Mesh를 유지한 채, Jaeger 관련 리소스만 삭제했습니다.

### 4.1 Observability 네임스페이스 삭제
```bash
kubectl delete namespace observability
```
이 명령은 Jaeger Operator, Jaeger 인스턴스, 인증서 등 모든 관련 리소스를 포함하여 `observability` 네임스페이스를 제거했습니다.

---

