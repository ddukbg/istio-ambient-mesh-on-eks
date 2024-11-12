## **EKS 환경에서 Istio Ambient Mesh 설치 가이드**

### **전제 조건**

1. **Kubernetes 클러스터 준비**: EKS 클러스터가 이미 생성되어 있어야 합니다.
   - Kubernetes 버전은 **1.28**, **1.29**, **1.30**, **1.31** 중 하나여야 합니다.

2. **로컬 환경 준비**:
   - `kubectl`이 설치되어 있어야 합니다.
   - **Helm 3.6** 이상이 설치되어 있어야 합니다.
   - AWS CLI가 설치 및 구성되어 있어야 합니다.

---

### **1. Istio CLI 다운로드 및 환경 설정**

Istio는 `istioctl`이라는 명령줄 도구를 사용하여 설치 및 관리됩니다.

1. **Istio 다운로드**:

   ```bash
   $ curl -L https://istio.io/downloadIstio | sh -
   ```

2. **Istio 디렉토리로 이동 및 PATH 설정**:

   ```bash
   $ cd istio-1.24.0
   $ export PATH=$PWD/bin:$PATH
   ```

   - `istio-1.24.0` 디렉토리로 이동합니다.
   - `istioctl`을 PATH에 추가하여 어디서든 명령을 실행할 수 있도록 설정합니다.

3. **Istioctl 버전 확인**:

   ```bash
   $ istioctl version
   ```

   - 출력 예시:

     ```
     no ready Istio pods in "istio-system"
     1.24.0
     ```

   - 현재 클러스터에 Istio가 설치되지 않았으므로 "no ready Istio pods" 메시지가 나타날 수 있습니다.

---

### **2. Istio Ambient Mode 설치**

Ambient Mesh를 설치하기 위해서는 `ambient` 프로파일을 사용해야 합니다.

1. **Istio 설치**:

   ```bash
   $ istioctl install --set profile=ambient --skip-confirmation
   ```

   - 설치 과정에서 별도의 확인 없이 진행되며, 설치 완료 시 다음과 같은 메시지가 출력됩니다.

     ```
     ✔ Istio core installed
     ✔ Istiod installed
     ✔ CNI installed
     ✔ Ztunnel installed
     ✔ Installation complete
     ```

2. **설치 확인**:

   ```bash
   $ istioctl verify-install
   ```

   - 설치된 구성 요소들이 정상적으로 배포되었는지 확인합니다.

---

### **3. Kubernetes Gateway API CRD 설치**

Istio는 Kubernetes Gateway API를 사용하여 트래픽 라우팅을 구성합니다. 대부분의 Kubernetes 클러스터에는 기본적으로 Gateway API CRD가 설치되어 있지 않으므로, 수동으로 설치해야 합니다.

```bash
$ kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml; }
```

- 이미 설치되어 있는 경우 이 단계를 건너뛰어도 됩니다.

---

### **4. EKS 환경에 대한 추가 설정**

EKS 환경에서 **Amazon VPC CNI**를 사용하고, **Pod ENI Trunking**이 활성화되어 있으며, **SecurityGroupPolicy**를 통해 Pod에 Security Group을 적용하고 있다면 추가 설정이 필요합니다.

#### **4.1. Pod ENI Trunking 활성화 여부 확인**

```bash
$ kubectl set env daemonset aws-node -n kube-system --list | grep ENABLE_POD_ENI
```

- `ENABLE_POD_ENI` 환경 변수가 설정되어 있으면 Pod ENI Trunking이 활성화된 것입니다.

#### **4.2. Pod-attached Security Groups 사용 여부 확인**

```bash
$ kubectl get securitygrouppolicies.vpcresources.k8s.aws
```

- 출력 결과가 있으면 Pod에 Security Group이 적용되어 있는 것입니다.

#### **4.3. POD_SECURITY_GROUP_ENFORCING_MODE 설정**

Istio는 kubelet 헬스 프로브에 link-local 주소를 사용하기 때문에, Amazon VPC CNI가 이를 인지하지 못해 헬스 체크가 실패할 수 있습니다. 이를 해결하기 위해 `POD_SECURITY_GROUP_ENFORCING_MODE`를 `standard`로 설정해야 합니다.

```bash
$ kubectl set env daemonset aws-node -n kube-system POD_SECURITY_GROUP_ENFORCING_MODE=standard
```

- 이 설정을 적용한 후, 영향을 받는 Pod들을 재시작해야 합니다.

---

### **5. Helm을 통한 Istio 설치 (권장 방법)**

생산 환경에서는 Helm을 사용하여 Istio를 설치하고 관리하는 것이 좋습니다. Helm을 사용하면 업그레이드 및 구성 관리가 용이합니다.

#### **5.1. Helm 저장소 추가 및 업데이트**

```bash
$ helm repo add istio https://istio-release.storage.googleapis.com/charts
$ helm repo update
```

#### **5.2. Istio Base Chart 설치**

Istio의 기본 CRD 및 클러스터 역할을 설치합니다.

```bash
$ helm install istio-base istio/base -n istio-system --create-namespace --wait
```

#### **5.3. Kubernetes Gateway API CRD 설치 (이미 수행했으면 건너뜀)**

```bash
$ kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml; }
```

#### **5.4. Istiod 컨트롤 플레인 설치**

```bash
$ helm install istiod istio/istiod --namespace istio-system --set profile=ambient --wait
```

- `istiod`는 Istio의 컨트롤 플레인으로, Ambient Mesh 모드를 사용하기 위해 `profile=ambient`로 설정합니다.

#### **5.5. Istio CNI 설치**

Istio CNI는 Ambient Mesh에서 트래픽 리다이렉션을 담당합니다.

```bash
$ helm install istio-cni istio/cni -n istio-system --set profile=ambient --wait
```

#### **5.6. ztunnel DaemonSet 설치**

`ztunnel`은 Ambient Mesh의 노드 프록시 역할을 합니다.

```bash
$ helm install ztunnel istio/ztunnel -n istio-system --wait
```

#### **5.7. 인그레스 게이트웨이 설치 (선택 사항)**

인그레스 게이트웨이가 필요한 경우 설치합니다.

```bash
$ helm install istio-ingress istio/gateway -n istio-ingress --create-namespace --wait
```

- 클러스터에서 LoadBalancer 타입의 서비스에 외부 IP가 할당되지 않는다면 `--wait` 옵션을 제거하여 무한 대기를 방지합니다.

---

### **6. 설치 확인**

#### **6.1. Helm 배포 상태 확인**

```bash
$ helm ls -n istio-system
```

- 출력 예시:

  ```
  NAME            NAMESPACE       REVISION    UPDATED                                 STATUS      CHART           APP VERSION
  istio-base      istio-system    1           2024-11-12 10:00:00 +0000 UTC          deployed    base-1.24.0     1.24.0
  istio-cni       istio-system    1           2024-11-12 10:01:00 +0000 UTC          deployed    cni-1.24.0      1.24.0
  istiod          istio-system    1           2024-11-12 10:02:00 +0000 UTC          deployed    istiod-1.24.0   1.24.0
  ztunnel         istio-system    1           2024-11-12 10:03:00 +0000 UTC          deployed    ztunnel-1.24.0  1.24.0
  ```

#### **6.2. Pod 상태 확인**

```bash
$ kubectl get pods -n istio-system
```

- 출력 예시:

  ```
  NAME                             READY   STATUS    RESTARTS   AGE
  istio-cni-node-abc123            1/1     Running   0          5m
  istiod-xyz456                    1/1     Running   0          5m
  ztunnel-def789                   1/1     Running   0          5m
  ```

- 모든 Pod가 `Running` 상태이고 재시작 없이 정상 동작하고 있는지 확인합니다.

---

### **7. 샘플 애플리케이션 배포 및 검증**

Istio Ambient Mesh가 정상적으로 동작하는지 확인하기 위해 샘플 애플리케이션을 배포합니다.

1. **샘플 애플리케이션 배포 가이드 참조**:

   - [Istio 공식 문서의 샘플 애플리케이션 배포 가이드](https://istio.io/latest/docs/setup/getting-started/#deploy-the-sample-application)를 따라 애플리케이션을 배포합니다.

2. **Ambient Mesh에 애플리케이션 추가**:

   - 네임스페이스에 레이블을 추가하여 애플리케이션을 Ambient Mesh에 포함시킵니다.

     ```bash
     $ kubectl label namespace <your-namespace> istio.io/dataplane-mode=ambient
     ```

3. **애플리케이션 동작 확인**:

   - 애플리케이션이 정상적으로 통신하고, Istio의 기능이 적용되는지 확인합니다.

---

### **8. 네트워크 폴리시 설정**

EKS 환경에서 네트워크 폴리시를 사용하고 있으므로, Ambient Mesh 구성 요소들이 정상적으로 통신할 수 있도록 네트워크 폴리시를 업데이트해야 합니다.

#### **8.1. ztunnel과 애플리케이션 Pod 간 통신 허용**

- 각 애플리케이션 네임스페이스에 다음과 같은 네트워크 폴리시를 적용합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-ztunnel
  namespace: <your-namespace>
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: ztunnel
      ports:
        - protocol: TCP
          port: 15008
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: ztunnel
      ports:
        - protocol: TCP
          port: 15006
```

- `app: ztunnel` 라벨은 실제 ztunnel Pod의 라벨로 대체해야 합니다.

#### **8.2. ztunnel 간 통신 허용**

- `istio-system` 네임스페이스에 다음 네트워크 폴리시를 적용합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ztunnel-to-ztunnel
  namespace: istio-system
spec:
  podSelector:
    matchLabels:
      app: ztunnel
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: ztunnel
      ports:
        - protocol: TCP
          port: 15441
```

#### **8.3. 컨피그 서버와의 통신 허용**

- 애플리케이션 Pod가 초기화 시 컨피그 서버에 접근할 수 있도록 네트워크 폴리시를 설정합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-config-server
  namespace: <your-namespace>
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: <config-server-namespace>
          podSelector:
            matchLabels:
              app: config-server
      ports:
        - protocol: TCP
          port: <config-server-port>
```

- `config-server-namespace`, `config-server`, `config-server-port`는 실제 환경에 맞게 대체해야 합니다.

---

### **9. 네임스페이스에 Ambient Mesh 적용**

애플리케이션이 실행되는 네임스페이스에 Ambient Mesh를 적용하기 위해 레이블을 추가합니다.

```bash
$ kubectl label namespace <your-namespace> istio.io/dataplane-mode=ambient
```

- 이 레이블을 추가하면 해당 네임스페이스의 모든 Pod가 Ambient Mesh에 포함됩니다.

---

### **10. 검증 및 모니터링**

1. **애플리케이션 동작 확인**:

   - 애플리케이션이 정상적으로 통신하고 있는지 확인합니다.
   - 컨피그 서버로부터 설정값을 정상적으로 가져오는지 확인합니다.

2. **네트워크 폴리시 적용 확인**:

   - 네트워크 폴리시가 올바르게 적용되었는지 확인합니다.
   - 필요 시 로그를 확인하여 통신에 문제가 없는지 점검합니다.

3. **모니터링 도구 활용**:

   - Istio의 모니터링 도구(Kiali 등)를 사용하여 트래픽 흐름과 메쉬 상태를 시각화하고 모니터링합니다.

---

### **추가 고려사항**

- **포트 및 라벨 정확성**: 네트워크 폴리시를 작성할 때 사용되는 포트 번호와 라벨은 환경에 따라 다를 수 있으므로 실제 값으로 대체해야 합니다.
  - ztunnel과 웨이포인트의 라벨은 다음 명령어로 확인할 수 있습니다.

    ```bash
    $ kubectl get pods -n istio-system --show-labels
    ```

- **Pod 재시작 필요성**: 네트워크 폴리시를 적용한 후, 필요한 경우 애플리케이션 Pod를 재시작하여 변경 사항이 적용되도록 합니다.

- **문서화 및 팀 공유**: 설정한 내용과 절차를 문서화하여 팀 내에 공유하고, 향후 유지보수에 활용합니다.

---

### **문제 해결**

- **설치 중 오류 발생 시**: 로그를 확인하고, 필요한 경우 Istio 공식 문서나 커뮤니티에 문의합니다.
- **네트워크 통신 문제**: 네트워크 폴리시 설정을 재검토하고, 필요한 통신이 허용되어 있는지 확인합니다.
- **컨피그 서버 접근 문제**: 컨피그 서버의 네임스페이스와 라벨이 정확한지, 네트워크 폴리시에서 올바르게 지정되었는지 확인합니다.

---

### **삭제 방법 (Optional)**

Istio를 제거해야 하는 경우 다음 단계를 따릅니다.

1. **Istio Ingress Gateway 삭제 (설치한 경우)**:

   ```bash
   $ helm delete istio-ingress -n istio-ingress
   $ kubectl delete namespace istio-ingress
   ```

2. **ztunnel 삭제**:

   ```bash
   $ helm delete ztunnel -n istio-system
   ```

3. **Istio CNI 삭제**:

   ```bash
   $ helm delete istio-cni -n istio-system
   ```

4. **istiod 컨트롤 플레인 삭제**:

   ```bash
   $ helm delete istiod -n istio-system
   ```

5. **Istio Base Chart 삭제**:

   ```bash
   $ helm delete istio-base -n istio-system
   ```

6. **Istio CRD 삭제 (Optional)**:

   - Istio 설치 시 생성된 CRD를 삭제합니다.

     ```bash
     $ kubectl get crd -oname | grep --color=never 'istio.io' | xargs kubectl delete
     ```

7. **istio-system 네임스페이스 삭제**:

   ```bash
   $ kubectl delete namespace istio-system
   ```

---

## **결론**

EKS 환경에서 Istio Ambient Mesh를 설치하기 위해서는 몇 가지 추가적인 설정과 고려사항이 있습니다. 특히 네트워크 폴리시를 사용하는 경우, Ambient Mesh 구성 요소들이 원활하게 통신할 수 있도록 필요한 통신을 허용하는 폴리시를 설정해야 합니다.

