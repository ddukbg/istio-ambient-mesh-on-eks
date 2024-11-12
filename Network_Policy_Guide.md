## ** 네트워크 폴리시 설정 가이드**

### **목차**

1. [소개](#소개)
2. [네트워크 폴리시의 중요성](#네트워크-폴리시의-중요성)
3. [네트워크 폴리시 설정 방법](#네트워크-폴리시-설정-방법)
   - [ztunnel과 애플리케이션 Pod 간의 통신 허용](#ztunnel과-애플리케이션-pod-간의-통신-허용)
   - [ztunnel 간의 통신 허용](#ztunnel-간의-통신-허용)
   - [웨이포인트와의 통신 설정 (필요 시)](#웨이포인트와의-통신-설정-필요-시)
4. [네트워크 폴리시 적용 시 주의사항](#네트워크-폴리시-적용-시-주의사항)
5. [테스트 및 검증 방법](#테스트-및-검증-방법)
6. [참고 자료](#참고-자료)

### **소개**

이 문서에서는 EKS 환경에서 Istio Ambient Mesh를 도입할 때 필요한 **네트워크 폴리시(NetworkPolicy)** 설정에 대해 자세히 설명합니다. 네트워크 폴리시는 클러스터 내 Pod 간의 네트워크 트래픽을 제어하여 보안을 강화하고 통신을 관리하는 데 중요한 역할을 합니다.

### **네트워크 폴리시의 중요성**

- **보안 강화**: 불필요한 트래픽을 차단하여 잠재적인 보안 위협을 감소시킵니다.
- **통신 제어**: 서비스 간의 통신을 세밀하게 관리하여 의도하지 않은 접근을 방지합니다.
- **운영 안정성 향상**: 예상치 못한 네트워크 트래픽으로 인한 시스템 부하를 줄일 수 있습니다.

### **네트워크 폴리시 설정 방법**

#### **ztunnel과 애플리케이션 Pod 간의 통신 허용**

**이유**: Ambient Mesh에서 **ztunnel**은 각 노드에서 실행되며, 애플리케이션 Pod와 통신하여 트래픽을 처리합니다. 따라서 ztunnel과 애플리케이션 Pod 간의 통신을 허용해야 합니다.

**네트워크 폴리시 예시**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-ztunnel
  namespace: your-app-namespace
spec:
  podSelector: {}  # 모든 Pod에 적용
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              istio.io/ztunnel: "true"
      ports:
        - protocol: TCP
          port: 15008
  egress:
    - to:
        - podSelector:
            matchLabels:
              istio.io/ztunnel: "true"
      ports:
        - protocol: TCP
          port: 15006
```

#### **ztunnel 간의 통신 허용**

**이유**: ztunnel은 다른 노드의 ztunnel과 통신하여 서비스 간 트래픽을 전달합니다.

**네트워크 폴리시 예시**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ztunnel-to-ztunnel
  namespace: istio-system
spec:
  podSelector:
    matchLabels:
      istio.io/ztunnel: "true"
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              istio.io/ztunnel: "true"
      ports:
        - protocol: TCP
          port: 15441
```

#### **웨이포인트와의 통신 설정 (필요 시)**

**이유**: 웨이포인트(waypoint)는 L7 트래픽을 처리하며, 필요한 경우 설정합니다.

**네트워크 폴리시 예시**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-waypoint
  namespace: your-app-namespace
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              istio.io/waypoint: "true"
      ports:
        - protocol: TCP
          port: 15008
  egress:
    - to:
        - podSelector:
            matchLabels:
              istio.io/waypoint: "true"
      ports:
        - protocol: TCP
          port: 15008
```


### **네트워크 폴리시 적용 시 주의사항**

- **정확한 라벨 사용**: ztunnel, 웨이포인트, 컨피그 서버의 실제 라벨을 확인하여 적용합니다.
- **포트 번호 확인**: 환경에 따라 사용되는 포트가 다를 수 있으므로 정확한 포트 번호를 사용합니다.
- **네임스페이스 범위 확인**: 네트워크 폴리시는 네임스페이스 단위로 적용되므로 필요한 네임스페이스에 올바르게 적용해야 합니다.
- **정책 우선순위**: 여러 네트워크 폴리시가 적용될 경우 우선순위를 고려합니다.

### **테스트 및 검증 방법**

- **통신 확인**:

  ```bash
  $ kubectl exec -it your-pod -n your-app-namespace -- curl http://config-server:port
  ```

- **네트워크 폴리시 시뮬레이션 도구 사용**: `kubectl`의 `--dry-run` 옵션이나 `netpol` 등의 도구를 사용하여 정책을 검증합니다.

- **로그 확인**: 통신 오류 시 애플리케이션 및 Istio 로그를 확인하여 문제를 진단합니다.

### **참고 자료**

- [Istio 공식 문서 - 앰비언트 네트워크 폴리시](https://istio.io/latest/docs/ambient/usage/networkpolicy/)
- [Kubernetes 공식 문서 - 네트워크 폴리시](https://kubernetes.io/ko/docs/concepts/services-networking/network-policies/)

