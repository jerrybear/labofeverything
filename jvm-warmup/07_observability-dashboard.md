# JVM Warmup 운영 대시보드와 관측 체크리스트

## 이 문서의 목적

이 문서는 JVM warmup 문제를 운영 관점에서 보려 할 때 어떤 대시보드 패널과 알림 항목이 필요한지 정리한 문서다.

## 핵심 원칙

- 평균값 하나로 보지 않는다.
- `시간 흐름` 기준으로 본다.
- `기동`, `웜업`, `정상 상태`를 같은 차트에 섞어 해석하지 않는다.
- 새 인스턴스 투입 이벤트와 성능 지표를 함께 본다.

## 반드시 있어야 할 대시보드 섹션

### 1. 인스턴스 라이프사이클 섹션

- process start timestamp
- readiness 통과 시점
- traffic attached 시점
- pod restart count
- deployment rollout 시점

이 섹션이 있어야 `언제부터 느렸는지`가 아니라 `어느 단계에서 느렸는지`를 볼 수 있다.

### 2. 사용자 체감 성능 섹션

- p50 latency
- p95 latency
- p99 latency
- request rate
- error rate

특히 첫 1분, 첫 5분 구간을 별도 필터로 볼 수 있으면 좋다.

### 3. JVM/프로세스 섹션

- CPU usage
- process start time
- heap used / max
- GC pause time
- allocation rate
- class loading count
- thread count

startup 직후 CPU spike와 allocation rate는 warmup 해석에 특히 유용하다.

### 4. 스케일링 섹션

- replica count
- HPA scale out/in event
- node scheduling delay
- pod pending to running time
- pod running to ready time

이 섹션이 없으면 startup CPU spike가 스케일 정책을 흔드는지 보기 어렵다.

### 5. 애플리케이션 초기화 섹션

- DB pool active/idle/creation metrics
- external HTTP client error/latency
- cache hit ratio
- 대표 쿼리 latency
- 대표 API cold path latency

JIT보다 애플리케이션 초기화가 더 큰 원인인 경우를 잡기 위한 섹션이다.

## 추천 패널 구성

### 패널 A: New Instance First 5 Minutes

- 인스턴스 생성 후 5분 동안의 p95/p99 latency
- CPU usage
- request rate
- error rate

새 인스턴스가 실제 사용자 경험에 어떤 영향을 주는지 가장 직관적으로 보여준다.

### 패널 B: Ready But Not Yet Stable

- readiness 통과 이후 0~60초 latency 분포
- steady-state 대비 배수 비교

`ready = stable`이 아님을 확인하는 데 유용하다.

### 패널 C: Startup vs Warmup 분리 패널

- process start -> ready 시간
- ready -> stable throughput 시간
- ready -> stable p99 시간

이 패널은 운영 토론에서 가장 유용한 기준점이 된다.

### 패널 D: Scale Event Overlay

- scale out 이벤트
- replica count
- p99 latency
- CPU usage

오토스케일이 문제를 해결하는지, 오히려 증폭하는지 판단할 수 있다.

## 알림 기준 예시

### 즉시 알림 후보

- deployment 후 첫 5분 p99가 기준치 초과
- 새 replica 추가 후 error rate 증가
- readiness는 성공했지만 첫 1분 latency가 평소 대비 급등

### 추세 알림 후보

- 최근 배포들에서 time to stable latency가 점점 증가
- 특정 서비스만 class loading count 또는 startup CPU가 비정상 증가
- pre-warming 적용 후에도 cold path latency 개선이 없음

## 해석 팁

- p50은 정상인데 p99만 나쁘면 tail latency 중심 문제일 가능성이 크다.
- CPU가 높고 처리량이 같이 좋아지면 JIT warmup일 가능성이 있다.
- CPU는 평범한데 첫 요청만 느리면 초기화/캐시 문제일 수 있다.
- scale out 직후 전체 latency가 더 나빠지면 새 인스턴스 warmup 비용이 서비스에 직접 노출되는 구조일 수 있다.

## 운영 체크리스트

1. 새 인스턴스의 첫 5분 지표를 따로 볼 수 있는가
2. readiness와 stable latency를 분리 측정하는가
3. scale out 이벤트를 latency 차트에 겹쳐 볼 수 있는가
4. p95/p99를 기본 패널로 보고 있는가
5. DB pool, cache, external dependency 초기화 지표를 함께 보는가

## 팀용 결론

JVM warmup 관측의 핵심은 `JVM 내부 지표를 더 많이 보는 것`이 아니라, `새 인스턴스가 사용자 성능에 미치는 초기 5분 영향`을 한눈에 보이게 만드는 것이다.
