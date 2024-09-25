## 기타
- 고통과 좌절에 대한 경험 공유

## Kubernetes 10년, 그 너머의 항해 : 살아남은 자의 인사이트
- k8s 기능들을 제대로 알고 있어야 한다 아니면 production 에서 쓰일 수 있을 정도로 제대로 올릴 수 없다
  - basic functions
    - scheduling
    - probe
    - replicas
    - storage
    - ingress controller
- AI on k8s
  - 2025~26 중요하다
- 기본 기술이 중요함
  - 앞서나가는 힘
- 지속적인 학습
  - 2~3년 전문가
- 운이 필요함
  - 커뮤니티 참석 -> 새로운 기술

---

## k8sgpt
### CNCF observability sandbox project
- 왜 필요한가?
  - k8s는 현대 인프라에서 표준이다
  - 사용하면 아름다워보이지만 실상은 다르다
  - k8s는 어렵다

- 동작 방식
  1. CrashLoopBackOff
  2. k8sgpt analyze
  3. k8sgpt analyze --explain
  4. solution ( cli, server 두 가지 형태 )
  5. (optional) --interactive mode

- 데모
  1. pvc를 띄움
  2. 7일동안 pending 상태
  3. k8sgpt analyze -n default
  4. k8sgpt analyze --explain

- 두뇌파의 공격 = PR : Remove Trash

- Slack 메시지 연동 - 노드 알림
  - custom logic
  - prometheus metric

- Security
  - pod 이름을 익명화해서 LLM으로 넘김
  - 만약 별로라면 OLLAMA를 사내망에서 운영

- 아이디어
  - k8sgpt 서버 구축(?)
  - cli를 먼저 사용해보고 직접 문제 상황에서 제 기능을 하는 지 검증

---

## 쿠버네티스 스케줄러는 노드를 어떻게 선택하는가?
- 파드가 배포될 적합한 노드를 결정
- 파드에 정의한 노드 필터링, 스코어링
  - 필터링 플러그인에 파드와 노드를 입력해서 조건을 만족하는 지 검사
  - 노드 목록이 비어있다면 스케줄링 불가 (=pending)
  - 모든 노드를 검사하지 않게 해주는 설정 값이 있음
  - 필터링을 통과한 노드들을 대상으로 점수 계산 및 순위 정렬 (weight)
  - [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
    - requiredDuringSchedulingIgnoreDuringExecution : 충족해야만 함 
    - preferredDuringSchedulingIgnoredDuringExecution : 선호도에 따라 다른 노드로 넘어감
  - 노드에서 파드 제외 - [Taints & Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

---

## 멀티리전 HA 구성
- LoxiLB 홍보느낌
- Disaster Recovery
- 멀티 클라우드 환경의 DR 표준화
  - multi AZ / multi region / multi cloud / onPrem
- SkyTV UK DR 전략
  - warm standby for multi cloud ( eks, gke )
    - warm standby 는 뭘 의미하는 것?
    - multi cloud 선택 이유는 aws 전체가 다운됐을 때의 재난 상황을 가정하는 것 인가?

---

## OpenTelemetry 기반 멀티 클러스터 Kubernetes Observability 구축

- 6개의 클러스터 약 300개 이상의 서비스
- Observability란
  - Monitoring
    - 시스템의 상태를 지속적으로 감시
    - 사전 정의된 Metric
    - 사후 대응 ( 수동적 )
  - Observability
    - 시스템의 상태를 지속적으로 감시
    - 시스템에서 예상치 못한 문제를 탐지하고 분석
    - Log(어플리케이션 내에서 발생하는 이벤트와 상태 변경을 기록한 시간 기반 데이터)
    - Metric(시스템 성능을 수치화한 특정 지표나 통계적인 데이터)
    - Trace(요청이 처리될 때 시스템 내에서 어떻게 처리되는 지 추적하는 데이터)
    - 사전/사후 대응 ( 능동적 )
- 기존 Observability 운영 환경 / 부족한 부분
- OpenTelemetry 와 Grafana Stack으로 통합된 Observability 환경 구성
- OpenTelemetry 도입 시 좋은 점
- OpenTelemetry 구성 요소
- 바뀐 Observability 운영 환경
  - Alloy
  - Mimir
  - Tempo
  - OpenTelemetry

- 궁금했던 점
  - 6개의 클러스터, 300개의 서비스를 운영할 때 istiod의 replicas 개수는 몇개 정도 되는지..?
  - proxy container의 어떤 로그를 수집하는지?
    - 그리고 grafana에서 알림 설정을 통해 대응하는지?
  - 바뀐 Observability 운영 환경 구성도에는 kiali, Jaeger가 없던데 사용하지 않는지?

---
## 얼마까지 알아보고 오셨어요? Kubernetes 비용 최적화
- Auto Scaling
  - pod
    - HPA
    - VPA
  - Node
    - Cluster AutoScaler
    - scale down utilization threshold
    - 단독 파드는 예외
- Right sizing
  - `resources.request` = 비용
    - 값이 클 경우, 불필요한 node로 인한 낭비 발생
  - Pod
    - VPA Recommender
    - kubecost request sizing recommendations
      - 3일 간의 데이터 중 max 기준으로 제안
    - 자동으로 HPA에 명시된 `resources.request` 값을 수정해주는 것 같음
    - limit 값도 함께 조정되어야 하기 때문에 참고
  - Node
    - kubecost - cluster sizing recommendations
- 미사용 리소스 중단
  - kube downscaler
- 가성비 향상
  - 지속적으로 k8s 업데이트 하기
  - 클라우드 정책 활용 (가격 정책 및 가성비 좋은 인프라)
- 지속 가능한 비용 최적화 환경
  - 측정
    - 계속해서 scale out 되면 reserved 되지 않은 cpu, mem이 있는 지
  - 감시
    - namespace 효율 감시
    - 비효율 workload top 20 감시
    - node의 효율 감시
  - 적용효과
    - 개발, 상용 환경
- 단일 클러스터에 kubecost 배포
