# CH 5. Introducing Envoy

# Introducing Envoy

## 서비스 프록시

- 서비스 대신 클라이언트 측 중계 요청을 전달
- 애플리케이션은 메서드 호출로 채널을 통해 메시지를 주고받을 수 있음
- 서비스 프록시는 투명하게 삽입되며 애플리케이션이 서비스 간 호출을 수행할 때 데이터 플레인의 존재 인식 X

## Iptables

- 리눅스의 호스트 기반 방화벽 및 패킷 조작 관리 도구
- 테이블, 체인, 규칙으로 구성
- 이스티오가 트래픽을 Envoy로 리디렉션하는 데 사용

## Envoy 프록시

- 오픈소스 애플리케이션 수준 서비스 프록시
- Lyft사에서 개발, 클라우드 네이티브 생태계에서 널리 사용

### Envoy 선택 이유

- 사이드카 배포에 적합한 설계
- 핫 리로드와 핫 재시작 기능
- API 기반의 런타임 구성
- ADS(Aggregated Discovery Service) 제공
- HTTP/2 및 gRPC 지원
    - HTTP/2의 멀티플렉싱 능력으로 성능 향상
    - gRPC 기본 지원 및 gRPC-JSON 트랜스코딩 기능

### Istio의 Envoy 사용

- 메시의 기본 단위로 투명하게 작동
- 애플리케이션 서비스에 사이드카로 배포
- 비루트 권한으로 실행

# Telemetry

마이크로 서비스간의 네트워크 연결성을 추적하고 기록하는 Telemetry를 제공한다.

- 메시 수준 - 작업 선택기가 없는 Istio 설치의 루트 구성 네임스페이스에 리소스를 배치
- 네임스페이스 - 작업 선택기가 없는 단일 Telemetry 리소스만 유효
- 작업 선택기가 있는 리소스의 경우, 주어진 작업을 선택하는 하나의 리소스만 유효

Telemetry 계층 구조

1. 작업별 구성
2. 네임스페이스별 구성
3. 루트 네임스페이스 구성

트래픽의 10%에 대해 무작위 샘플링을 활성화하는 정책의 경우 다음과 같이 구성될 수 있다:

```yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  # no selector specified, applies to all workloads
  tracing:
  - randomSamplingPercentage: 10.00
```