# Spring Boot JVM Warmup 상세 대응 가이드

## 이 문서의 목적

이 문서는 `06_spring-boot-playbook.md`의 `대응 순서`와 `Spring Boot에서 자주 유용한 실행 아이디어`를 실제 실행 단위로 더 구체화한 문서다. 목표는 `무엇을 볼지`에서 끝나지 않고 `무엇을 어떤 순서로 해볼지`까지 바로 이어지게 하는 것이다.

## 먼저 전제부터

- 이 문서는 `Spring Boot` 서비스가 이미 운영 중이거나 배포 예정인 상황을 전제로 한다.
- 문제는 보통 `readiness는 통과했는데 첫 요청이 느리다`, `새 Pod 붙을 때 p99가 튄다`, `CPU throttling과 함께 초반 지연이 길어진다` 형태로 보인다.
- 핵심은 `JVM warmup`, `Spring Boot 초기화`, `운영 정책`을 한 세트로 다루는 것이다.

## 대응 순서 상세 가이드

### 1단계: 현상을 먼저 분해하기

가장 먼저 해야 할 일은 `느리다`를 더 작은 질문으로 나누는 것이다.

#### 확인 질문

- 느린 시점이 `process start 직후`인가
- 느린 시점이 `readiness 직후`인가
- 느린 요청이 `첫 1회` 중심인가, `첫 1분` 전체인가
- 모든 API가 느린가, 특정 API만 느린가
- 새 Pod가 붙을 때만 느린가, 재시작 없이도 주기적으로 느린가

#### 이 단계의 산출물

- startup time
- readiness 이후 첫 1분 p95/p99
- 대표 API 3~5개별 cold/warm latency 비교
- 새 Pod 투입 시점과 latency spike 연관성

이 산출물이 있어야 뒤 단계에서 `JIT 문제`, `초기화 문제`, `운영 정책 문제`를 나눠 볼 수 있다.

### 2단계: warmup 대상 경로를 고르기

아무 엔드포인트나 예열하면 효과가 약하다. 실제로 사용자 지연에 영향을 주는 경로를 골라야 한다.

#### 우선순위 기준

1. 호출량이 높은 API
2. 첫 요청 지연이 특히 긴 API
3. 인증/인가 필터를 반드시 타는 API
4. DB 쿼리, 외부 HTTP 호출, 직렬화 비용이 큰 API
5. 캐시 초기화나 템플릿/매퍼 준비가 숨어 있는 API

#### 대표 warmup 후보 예시

- `/api/users/me` 같은 대표 인증 API
- 메인 목록 조회 API
- 가장 무거운 JPA repository query가 포함된 API
- JSON 응답 구조가 큰 DTO API
- 외부 결제/회원/상품 API 연동이 붙는 유스케이스

#### 피해야 할 선택

- 헬스체크 엔드포인트만 호출
- 실제 트래픽과 거의 상관없는 관리성 API만 호출
- DB도 외부 의존성도 안 타는 지나치게 가벼운 API만 호출

### 3단계: 병목이 JVM 쪽인지 Spring Boot 초기화 쪽인지 가설 세우기

이 단계에서는 `무엇을 먼저 의심할지`를 정리한다.

#### JVM warmup 쪽 신호

- startup 직후 CPU가 높고 시간이 지나며 처리량이 점진적으로 좋아짐
- 첫 수십 초 p99가 높다가 안정화됨
- 트래픽이 충분히 들어올 때 성능이 개선됨

#### Spring Boot 초기화 쪽 신호

- 특정 API 첫 호출만 유난히 느림
- 첫 DB 쿼리, 첫 외부 호출, 첫 직렬화만 느림
- readiness는 빨리 통과했는데 실제 유스케이스 호출에서만 지연 발생
- 캐시/커넥션 풀/매퍼 준비 후 바로 안정화됨

#### 운영 정책 쪽 신호

- 새 Pod마다 같은 현상 반복
- CPU limit이 낮은 환경에서만 심함
- scale out 직후 전체 latency가 더 악화됨
- CPU throttling 지표와 latency spike가 같이 나타남

### 4단계: pre-warming 설계를 한다

pre-warming은 `무엇을`, `언제`, `어디서`, `얼마나` 할지 정해야 효과가 있다.

#### 무엇을 예열할까

- 대표 API 호출
- 대표 DB query 실행
- 대표 DTO 직렬화/역직렬화
- 외부 HTTP client DNS/TLS/connection 준비
- 핵심 캐시 키 일부 로딩

#### 언제 할까

- application startup 직후
- readiness 통과 직후, 실제 트래픽 연결 전
- 배포 파이프라인에서 새 Pod를 트래픽에 붙이기 전

#### 어디서 할까

- 애플리케이션 내부 `ApplicationRunner`
- 별도 warmup service/component
- sidecar 또는 배포 파이프라인의 test traffic 단계

#### 얼마나 할까

- 너무 적으면 효과가 없음
- 너무 많으면 startup 부하만 키움
- 보통은 `대표 경로를 소수 회수로 태우는 방식`이 출발점으로 적당함

### 5단계: 첫 요청에 몰린 비용을 앞당긴다

아래 항목은 Spring Boot에서 자주 숨어 있는 비용들이다.

#### DB와 ORM

- JPA/Hibernate 메타모델 준비
- 첫 쿼리 실행 시 SQL generation/plan/cache 준비
- 커넥션 풀 초기 connection 확보

#### HTTP/네트워크

- 외부 API 대상 DNS lookup
- TLS handshake
- connection pool 초기 연결

#### 직렬화와 메시지 처리

- Jackson serializer/deserializer 준비
- 큰 DTO 구조의 reflection/metadata 준비
- 메시지 컨버터 첫 초기화

#### 보안과 프록시

- 인증/인가 필터 체인 첫 통과
- AOP proxy 경로 첫 실행
- method security 관련 준비

이 비용은 실제 요청에서 처음 맞게 두기보다, 예열 경로에서 한 번 미리 맞게 하는 것이 보통 낫다.

### 6단계: CPU throttling 가능성을 같이 점검한다

웜업 장애가 심해지는 대표 패턴이다.

#### 자주 보이는 흐름

1. 새 Pod 시작
2. JVM warmup + Spring 초기화가 동시에 CPU 사용 증가
3. CPU limit에 걸려 throttling 발생
4. warmup과 초기화가 더 길어짐
5. readiness 이후에도 응답 지연 지속
6. 오토스케일 또는 재시도 트래픽으로 더 악화

#### 같이 봐야 할 항목

- container CPU usage
- CPU throttling 지표
- startup 직후 p99 latency
- scale out 이벤트
- pod ready 이후 트래픽 유입 시점

#### 실무 대응 방향

- startup 직후 CPU 여유 확보
- 너무 낮은 CPU limit 완화 검토
- 새 Pod에 즉시 전체 트래픽을 붙이지 않기
- HPA가 startup CPU spike를 과민하게 반영하지 않는지 확인

### 7단계: 구현 패턴을 고른다

#### 패턴 A: ApplicationRunner 기반 간단 예열

언제 적합한가:

- 애플리케이션 내부에서 간단히 예열 가능할 때
- 대표 API/쿼리/직렬화 경로가 명확할 때

장점:

- 구현이 단순함
- 앱 내부 문맥을 바로 활용 가능함

주의점:

- startup 시간을 과도하게 늘릴 수 있음
- readiness 전에 돌리면 기동 시간이 길어질 수 있음

#### 패턴 B: readiness 이후 내부 warmup job

언제 적합한가:

- readiness는 먼저 열되 실제 트래픽 연결 전 예열 구간을 둘 수 있을 때

장점:

- 기동과 예열을 논리적으로 분리 가능
- warmup 진행 상태를 따로 관리하기 좋음

주의점:

- `ready`와 `safe to receive traffic` 구분이 필요함

#### 패턴 C: 배포 파이프라인에서 test traffic 주입

언제 적합한가:

- Kubernetes/service mesh/ingress 제어가 가능할 때
- 새 인스턴스를 트래픽에 붙이기 전 별도 검증 단계가 있을 때

장점:

- 운영 흐름과 잘 맞음
- 사용자 트래픽과 분리된 예열 가능

주의점:

- 파이프라인 복잡도가 올라감
- 예열 트래픽이 실제 hot path를 잘 반영해야 함

## Spring Boot에서 자주 유용한 실행 아이디어 상세

### 아이디어 1: 대표 API 3종 예열

- 인증 API 1개
- 조회 API 1개
- 외부 의존 또는 DB 비용 큰 API 1개

이 조합은 필터 체인, 직렬화, DB/외부 의존 초기화를 넓게 건드릴 수 있어 출발점으로 좋다.

### 아이디어 2: JPA query 전용 예열

- 실제로 가장 무거운 repository query 몇 개를 startup 또는 warmup job에서 한 번 실행
- N+1, fetch graph, projection, converter 관련 초기 비용을 미리 노출

특히 `첫 쿼리만 유난히 느린 서비스`에서 효과를 보기 쉽다.

### 아이디어 3: Jackson/DTO 직렬화 예열

- 응답 크기가 큰 DTO를 대표 객체로 직렬화 한 번 수행
- JSON schema/serializer 준비 비용을 앞당김

API 로직은 빠른데 첫 응답만 느린 경우 유용하다.

### 아이디어 4: 외부 연동 client 예열

- WebClient/Feign/RestTemplate 기반 외부 연동 대상에 가벼운 검증 요청 수행
- DNS, TLS, connection pool, timeout 경로 초기 비용 점검 가능

단, 실제 외부 시스템 부하를 고려해 너무 공격적으로 호출하면 안 된다.

### 아이디어 5: cache seed 최소 버전

- 모든 캐시를 다 채우려 하지 말고, 첫 1분 사용자 체감에 영향을 주는 핵심 키만 선적재
- 캐시 미스 폭탄을 줄이는 목적에 집중

### 아이디어 6: 인증/인가 경로 예열

- 인증 필터, 토큰 파서, 권한 체크가 무거운 경우 대표 보호 API를 예열 대상으로 포함
- 보안 경로가 빠지면 실제 사용자 요청과 warmup 경로가 어긋날 수 있다.

## 적용 순서 추천

가장 현실적인 적용 순서는 보통 아래와 같다.

1. 첫 1분 p99와 새 Pod 첫 5분 지표를 분리해 본다.
2. 대표 API 3종을 선정한다.
3. 간단한 pre-warming을 넣고 전후 비교한다.
4. 효과가 있으면 JPA/직렬화/외부연동 예열을 추가한다.
5. 그래도 심하면 CPU throttling, HPA, rollout 방식까지 함께 손본다.

## 피해야 할 실수

- 예열 대상을 너무 많이 잡아 startup만 더 무겁게 만드는 것
- readiness 전에 모든 예열을 몰아 넣는 것
- 실제 사용자 경로와 무관한 warmup만 수행하는 것
- CPU throttling을 무시하고 애플리케이션 로직만 보는 것
- steady-state 개선만 보고 cold/warm 상태 검증을 생략하는 것

## 한 줄 결론

Spring Boot에서 JVM warmup 대응을 실제로 성공시키려면 `대표 사용자 경로를 기준으로 예열 대상을 고르고`, `첫 요청 비용을 앞당기며`, `CPU throttling과 rollout 정책까지 함께 점검`해야 한다.
