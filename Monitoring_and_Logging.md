## ** 모니터링 및 로깅 설정 **

### **목차**

1. [소개](#소개)
2. [모니터링 도구 설치](#모니터링-도구-설치)
   - [Prometheus 설치](#prometheus-설치)
   - [Grafana 설치](#grafana-설치)
   - [Kiali 설치](#kiali-설치)
3. [분산 추적 설정](#분산-추적-설정)
   - [Jaeger 설치](#jaeger-설치)
4. [메트릭 및 로그 확인](#메트릭-및-로그-확인)
5. [참고 자료](#참고-자료)

### **소개**

이 문서에서는 Istio Ambient Mesh 환경에서 **모니터링**과 **로깅**을 설정하는 방법을 안내합니다. 이를 통해 시스템의 상태를 실시간으로 파악하고 문제를 조기에 발견할 수 있습니다.

### **모니터링 도구 설치**

#### **Prometheus 설치**

- **설치 명령어**:

  ```bash
  $ kubectl apply -f samples/addons/prometheus.yaml
  ```

- **상태 확인**:

  ```bash
  $ kubectl get pods -n istio-system -l app=prometheus
  ```

#### **Grafana 설치**

- **설치 명령어**:

  ```bash
  $ kubectl apply -f samples/addons/grafana.yaml
  ```

- **접속 방법**:

  ```bash
  $ kubectl port-forward svc/grafana -n istio-system 3000:3000
  ```

  브라우저에서 `http://localhost:3000`으로 접속합니다.

#### **Kiali 설치**

- **설치 명령어**:

  ```bash
  $ kubectl apply -f samples/addons/kiali.yaml
  ```

- **접속 방법**:

  ```bash
  $ kubectl port-forward svc/kiali -n istio-system 20001:20001
  ```

  브라우저에서 `http://localhost:20001/kiali`로 접속합니다.

### **분산 추적 설정**

#### **Jaeger 설치**

- **설치 명령어**:

  ```bash
  $ kubectl apply -f samples/addons/jaeger.yaml
  ```

- **접속 방법**:

  ```bash
  $ kubectl port-forward svc/jaeger-query -n istio-system 16686:16686
  ```

  브라우저에서 `http://localhost:16686`으로 접속합니다.

### **메트릭 및 로그 확인**

- **Prometheus**: 메트릭 수집 및 쿼리
- **Grafana**: 대시보드를 통해 메트릭 시각화
- **Kiali**: 서비스 메쉬의 구조와 트래픽 흐름 시각화
- **Jaeger**: 분산 추적을 통한 지연 시간 분석

### **참고 자료**

- [Istio 공식 문서 - 모니터링](https://istio.io/latest/docs/ops/monitoring/)
- [Prometheus 공식 사이트](https://prometheus.io/)
- [Grafana 공식 사이트](https://grafana.com/)
- [Kiali 공식 사이트](https://kiali.io/)
- [Jaeger 공식 사이트](https://www.jaegertracing.io/)
