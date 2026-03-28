# Spring Boot 기준 JVM Warmup 대응 플레이북

## 이 문서는 누구를 위한 것인가

이 문서는 `Spring Boot` 서비스에서 시작 직후 느림, 새 Pod 투입 직후 tail latency 상승, readiness는 통과했는데 실제 응답이 아직 불안정한 문제를 다루기 위한 실무용 플레이북이다.

## 먼저 결론부터

- Spring Boot에서는 JVM JIT만이 아니라 `자동 설정`, `빈 초기화`, `프록시`, `커넥션 풀`, `직렬화 준비`, `캐시 미적재`가 초반 성능에 같이 영향을 준다.
- 그래서 `JVM warmup 대응`은 JVM 옵션만 만지는 작업이 아니라 `애플리케이션 기동 방식 + readiness + 운영 정책`을 함께 손보는 작업이다.

## Spring Boot에서 특히 먼저 볼 것

### 1. readiness와 실제 서비스 가능 시점을 구분하기

- Actuator readiness가 통과했다고 바로 실제 트래픽을 받게 하면 초반 요청이 느릴 수 있다.
- readiness는 최소 조건이고, warmup 완료는 별도 운영 판단일 수 있다.
- 가능하면 readiness 직후 pre-warming 구간을 둔다.

### 2. 초기화 비용이 큰 컴포넌트 찾기

- JPA/Hibernate 메타모델 초기화
- Jackson ObjectMapper 첫 사용 구간
- WebClient/RestTemplate 대상 외부 연결 준비
- HikariCP 같은 커넥션 풀 초기 연결
- Redis, cache, template engine 초기화
- 보안 필터 체인, AOP 프록시, reflection-heavy 구성

### 3. 첫 요청에만 몰린 비용 찾기

- 특정 API 첫 호출 때만 느린지
- DB 쿼리 첫 실행이 느린지
- 직렬화/역직렬화 첫 구간이 느린지
- 템플릿 렌더링이나 메시지 컨버터 첫 사용이 느린지

## 대응 순서

### 1단계: 측정부터 분리하기

- process start -> readiness 시간 측정
- readiness 이후 첫 1분 p95/p99 측정
- 첫 요청 1회와 첫 100회 패턴 비교
- 새 Pod 추가 시점의 latency 변화 기록

이 단계를 빼고 바로 튜닝하면 `startup 문제`와 `warmup 문제`를 섞어서 보게 된다.

### 2단계: pre-warming 넣기

가장 효과가 좋은 경우가 많다.

- 실제 자주 쓰는 API를 내부적으로 몇 번 호출
- 자주 쓰는 DB 쿼리 1회 이상 실행
- 대표 직렬화/역직렬화 경로 실행
- 외부 의존 연결이 있다면 연결 성립 여부 확인
- 캐시가 있다면 핵심 키 일부 선적재 검토

중요한 점은 `헬스체크 엔드포인트만 호출`해서는 충분하지 않은 경우가 많다는 점이다.

### 3단계: 불필요한 초기화 줄이기

- 쓰지 않는 starter 제거
- 무거운 자동 설정 비활성화 검토
- 애플리케이션 시작 시 꼭 필요하지 않은 로직은 지연 초기화 또는 비동기화 검토
- reflection-heavy 라이브러리 사용 구간 점검
- 첫 요청에서만 발생하는 on-demand 준비 비용을 앞당길 수 있는지 확인

### 4단계: 운영 정책 손보기

- 최소 Pod 수 유지
- scale-to-zero 또는 과도한 pod churn 완화
- 배포 직후 트래픽을 서서히 붙이기
- HPA가 startup CPU spike를 과민하게 반영하지 않게 확인
- 짧은 배치/워커는 실행 시간을 늘리거나 묶어서 돌릴 수 있는지 검토

## Spring Boot에서 자주 유용한 실행 아이디어

### 예열 대상 후보

- 가장 호출량이 높은 GET API
- 가장 비싼 JPA repository query
- Jackson 직렬화가 무거운 DTO 경로
- 인증/인가 필터를 거치는 대표 요청
- 외부 HTTP 호출이 붙은 대표 유스케이스

### 예열 방식 후보

- ApplicationRunner 또는 별도 warmup component
- readiness 후 내부 pre-warming job 실행
- 배포 파이프라인에서 새 인스턴스에 테스트 트래픽 주입
- service mesh나 load balancer 연결 전에 내부 예열 수행

## 체크리스트

### 배포 전

- readiness와 warmup 완료를 구분했는가
- 대표 API와 쿼리를 선정했는가
- 첫 1분 p99를 대시보드에서 볼 수 있는가
- pre-warming 경로가 실제 hot path를 반영하는가

### 배포 중

- 새 Pod 투입 직후 latency spike를 보는가
- startup CPU spike가 HPA에 어떤 영향을 주는가
- 외부 의존성 연결 실패나 재시도가 warmup을 더 길게 만들지 않는가

### 배포 후

- 첫 1분 p95/p99가 개선됐는가
- steady-state만 좋아지고 cold/warm 상태는 그대로인 것은 아닌가
- pre-warming 자체가 과도한 부하를 만들지는 않는가

## 피해야 할 실수

- JVM 옵션만 바꾸면 해결될 거라고 보는 것
- readiness 통과만 기준으로 삼는 것
- local 개발환경에서 한 번 빨라졌다고 운영도 같다고 보는 것
- 평균 latency만 보고 tail latency를 놓치는 것

## 한 줄 결론

Spring Boot에서 JVM warmup 대응의 핵심은 `첫 요청에 몰리는 초기화 비용을 찾아 앞당기고`, `readiness와 실제 안정화 시점을 분리해 운영`하는 것이다.
