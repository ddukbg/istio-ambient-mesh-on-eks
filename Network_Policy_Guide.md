## **네트워크 폴리시 설정 가이드**

### **목차**

1. [소개](#소개)
2. [네트워크 폴리시의 중요성](#네트워크-폴리시의-중요성)
3. [네트워크 폴리시 설정 방법](#네트워크-폴리시-설정-방법)
   - [ztunnel과 애플리케이션 Pod 간의 통신 허용](#ztunnel과-애플리케이션-pod-간의-통신-허용)
   - [ztunnel 간의 통신 허용](#ztunnel-간의-통신-허용)
   - [웨이포인트와의 통신 설정 (필요 시)](#웨이포인트와의-통신-설정-필요-시)
   - [포트 15008 허용 및 링크 로컬 주소 처리](#포트-15008-허용-및-링크-로컬-주소-처리)
4. [네트워크 폴리시 적용 시 주의사항](#네트워크-폴리시-적용-시-주의사항)
5. [테스트 및 검증 방법](#테스트-및-검증-방법)
6. [참고 자료](#참고-자료)

### **소개**

이 문서에서는 **EKS 환경**에서 **Istio Ambient Mesh**를 도입할 때 필요한 **네트워크 폴리시(NetworkPolicy)** 설정에 대해 자세히 설명합니다. 네트워크 폴리시는 클러스터 내 Pod 간의 네트워크 트래픽을 제어하여 보안을 강화하고 통신을 관리하는 데 중요한 역할을 합니다. 특히, EKS 환경에서 **Amazon VPC CNI**와 함께 사용할 때 고려해야 할 사항들을 다룹니다.

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
  podSelector: {}  # 네임스페이스 내 모든 Pod에 적용
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
          port: 15008  # ztunnel에서 애플리케이션 Pod로의 트래픽 포트
  egress:
    - to:
        - podSelector:
            matchLabels:
              istio.io/ztunnel: "true"
      ports:
        - protocol: TCP
          port: 15006  # 애플리케이션 Pod에서 ztunnel로의 트래픽 포트
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
          port: 15441  # ztunnel 간의 통신 포트
```

#### **웨이포인트와의 통신 설정 (필요 시)**

**이유**: 웨이포인트(waypoint)는 L7 트래픽을 처리하며, 필요한 경우 설정합니다.

**네트워크 폴리시 예시 (웨이포인트를 사용하는 경우)**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-waypoint
  namespace: your-app-namespace
spec:
  podSelector: {}  # 네임스페이스 내 모든 Pod에 적용
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
          port: 15008  # 웨이포인트에서 애플리케이션 Pod로의 트래픽 포트
  egress:
    - to:
        - podSelector:
            matchLabels:
              istio.io/waypoint: "true"
      ports:
        - protocol: TCP
          port: 15008  # 애플리케이션 Pod에서 웨이포인트로의 트래픽 포트
```

#### **포트 15008 허용 및 링크 로컬 주소 처리**

**1. 포트 15008로의 인바운드 트래픽 허용**

**이유**: Ambient Mesh에서는 ztunnel이나 웨이포인트가 **애플리케이션 Pod로 트래픽을 전달할 때 포트 15008**을 사용합니다. **NetworkPolicy가 이 포트를 차단하면 메쉬 트래픽이 전달되지 않습니다.**

**네트워크 폴리시 예시**:

```yaml
# 기존에 특정 포트만 허용하던 NetworkPolicy를 수정하여 포트 15008을 추가
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-inbound-ports
  namespace: your-app-namespace
spec:
  podSelector:
    matchLabels:
      app: your-app
  policyTypes:
    - Ingress
  ingress:
    - ports:
        - protocol: TCP
          port: 80      # 기존에 허용하던 포트
        - protocol: TCP
          port: 443     # 기존에 허용하던 포트
        - protocol: TCP
          port: 15008   # Ambient Mesh를 위한 포트 추가
```

**2. 링크 로컬 주소(169.254.7.127/32)로부터의 인바운드 트래픽 허용**

**이유**: **kubelet의 헬스 프로브(Health Probe)**는 Ambient Mesh에서 **링크 로컬 주소(169.254.7.127)로부터 오는 트래픽으로 처리**됩니다. **NetworkPolicy가 이 주소로부터의 트래픽을 차단하면 헬스 프로브가 실패**합니다.

**네트워크 폴리시 예시**:

```yaml
# 링크 로컬 주소로부터의 트래픽을 허용하는 규칙 추가
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-health-probe
  namespace: your-app-namespace
spec:
  podSelector:
    matchLabels:
      app: your-app
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 169.254.7.127/32
```

### **네트워크 폴리시 적용 시 주의사항**

- **정확한 라벨 사용**: `ztunnel`, `웨이포인트`, `애플리케이션 Pod`의 실제 라벨을 확인하여 적용합니다.
  - 라벨 확인 방법:
    ```bash
    $ kubectl get pods -n istio-system --show-labels
    ```
- **포트 번호 확인**: 환경에 따라 사용되는 포트가 다를 수 있으므로 정확한 포트 번호를 사용합니다.
- **네임스페이스 범위 확인**: 네트워크 폴리시는 네임스페이스 단위로 적용되므로 필요한 네임스페이스에 올바르게 적용해야 합니다.
- **CNI 플러그인 고려**: **EKS에서는 Amazon VPC CNI를 사용하므로**, **링크 로컬 주소로부터의 트래픽 처리가 CNI 설정에 따라 다를 수 있습니다.**
  - **POD_SECURITY_GROUP_ENFORCING_MODE 설정**: 링크 로컬 주소로부터의 트래픽이 차단되지 않도록 `POD_SECURITY_GROUP_ENFORCING_MODE=standard`로 설정해야 합니다.
    ```bash
    $ kubectl set env daemonset aws-node -n kube-system POD_SECURITY_GROUP_ENFORCING_MODE=standard
    ```
- **정책 우선순위**: 여러 네트워크 폴리시가 적용될 경우 우선순위를 고려합니다.
- **헬스 프로브 트래픽 허용**: 링크 로컬 주소로부터의 트래픽을 허용하여 헬스 체크가 실패하지 않도록 합니다.

### **테스트 및 검증 방법**

- **통신 확인**:

  ```bash
  # 애플리케이션 Pod에서 ztunnel로의 통신 테스트
  $ kubectl exec -it your-pod -n your-app-namespace -- curl http://127.0.0.1:15006

  # 애플리케이션 Pod로의 인바운드 트래픽 테스트
  $ kubectl exec -it another-pod -n your-app-namespace -- curl http://your-app:80
  ```

- **헬스 프로브 상태 확인**:

  ```bash
  $ kubectl get pods -n your-app-namespace
  # 모든 Pod가 Running 상태인지 확인
  ```

- **네트워크 폴리시 시뮬레이션 도구 사용**: `np-tool` 또는 `kubectl`의 `--dry-run` 옵션을 사용하여 정책을 검증합니다.

- **로그 확인**: 통신 오류 시 애플리케이션 및 Istio 로그를 확인하여 문제를 진단합니다.

### **참고 자료**

- [Istio 공식 문서 - Ambient Mesh와 Kubernetes NetworkPolicy](https://istio.io/latest/docs/ops/deployment/ambient/network-policy/)
- [Kubernetes 공식 문서 - 네트워크 폴리시](https://kubernetes.io/ko/docs/concepts/services-networking/network-policies/)
- [AWS EKS 공식 문서 - Amazon VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html)
- [AWS 블로그 - EKS에서 네트워크 폴리시 사용하기](https://aws.amazon.com/ko/blogs/opensource/network-policies-eks/)

---

** Ambient Mesh와 NetworkPolicy를 함께 사용할 때 특히 **포트 15008 허용**과 **헬스 프로브 트래픽을 위한 링크 로컬 주소 허용**이 중요합니다.

이 가이드가 도움이 되시길 바라며, 추가적인 질문이나 도움이 필요하시면 언제든지 말씀해 주세요!
