## ** 샘플 애플리케이션 배포 및 테스트 **

**파일명**: `Sample_Application_Deployment.md`

### **목차**

1. [소개](#소개)
2. [샘플 애플리케이션 배포](#샘플-애플리케이션-배포)
3. [Ambient Mesh에 애플리케이션 추가](#ambient-mesh에-애플리케이션-추가)
4. [애플리케이션 동작 확인](#애플리케이션-동작-확인)
5. [mTLS 및 보안 기능 테스트](#mtls-및-보안-기능-테스트)
6. [트래픽 관리 기능 적용 및 테스트](#트래픽-관리-기능-적용-및-테스트)
7. [참고 자료](#참고-자료)

### **소개**

이 문서에서는 Istio Ambient Mesh의 기능을 테스트하기 위해 샘플 애플리케이션을 배포하고 검증하는 방법을 안내합니다. 샘플 애플리케이션으로는 Istio에서 제공하는 **Bookinfo** 애플리케이션을 사용합니다.

### **샘플 애플리케이션 배포**

1. **네임스페이스 생성**:

   ```bash
   $ kubectl create namespace bookinfo
   ```

2. **애플리케이션 배포**:

   ```bash
   $ kubectl apply -n bookinfo -f samples/bookinfo/platform/kube/bookinfo.yaml
   ```

3. **인그레스 게이트웨이 설정**:

   ```bash
   $ kubectl apply -n bookinfo -f samples/bookinfo/networking/bookinfo-gateway.yaml
   ```

### **Ambient Mesh에 애플리케이션 추가**

- **네임스페이스에 레이블 추가**:

  ```bash
  $ kubectl label namespace bookinfo istio.io/dataplane-mode=ambient
  ```

- **Pod 재시작**: 레이블 적용 후 기존 Pod를 재시작하여 Ambient Mesh에 포함시킵니다.

  ```bash
  $ kubectl rollout restart deployment -n bookinfo
  ```

### **애플리케이션 동작 확인**

- **인그레스 IP 확인**:

  ```bash
  $ kubectl get svc istio-ingress -n istio-ingress
  ```

- **애플리케이션 접속**:

  브라우저에서 다음 URL로 접속합니다.

  ```
  http://<INGRESS_IP>/productpage
  ```

- **정상 동작 확인**: 애플리케이션 페이지가 정상적으로 표시되는지 확인합니다.

### **mTLS 및 보안 기능 테스트**

- **mTLS 활성화 여부 확인**:

  ```bash
  $ kubectl get peerauthentication -n bookinfo
  ```

- **보안 정책 적용**:

  `AuthorizationPolicy`를 적용하여 특정 서비스에 대한 접근을 제어합니다.

  ```yaml
  apiVersion: security.istio.io/v1beta1
  kind: AuthorizationPolicy
  metadata:
    name: productpage-policy
    namespace: bookinfo
  spec:
    selector:
      matchLabels:
        app: productpage
    action: DENY
    rules:
      - from:
          - source:
              principals: ["*"]
  ```

- **정책 효과 확인**: 해당 서비스에 대한 접근이 차단되었는지 확인합니다.

### **트래픽 관리 기능 적용 및 테스트**

- **버전별 트래픽 분할**:

  ```yaml
  apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  metadata:
    name: reviews
    namespace: bookinfo
  spec:
    hosts:
      - reviews
    http:
      - route:
          - destination:
              host: reviews
              subset: v1
            weight: 80
          - destination:
              host: reviews
              subset: v2
            weight: 20
  ```

- **적용 결과 확인**: 애플리케이션을 새로 고침하며 버전별로 트래픽이 분산되는지 확인합니다.

### **참고 자료**

- [Istio 공식 문서 - Bookinfo 예제](https://istio.io/latest/docs/examples/bookinfo/)
- [Istio 공식 문서 - 트래픽 관리](https://istio.io/latest/docs/tasks/traffic-management/)

