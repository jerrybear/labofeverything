# JVM Warmup 이슈 측정과 대응

## 먼저 측정 기준을 나눠야 한다

웜업 이슈를 제대로 보려면 최소한 아래 세 가지를 분리해야 한다.

- `startup time`: 프로세스 시작부터 readiness까지
- `time to stable latency`: 지연 시간이 대체로 안정될 때까지
- `time to stable throughput`: 처리량이 대체로 안정될 때까지

이 셋을 하나의 평균 숫자로 섞으면 판단이 흐려진다.

## 흔한 측정 실수

- 첫 요청 몇 개만 보고 전체 성능이라고 말함
- 반대로 충분히 웜업된 뒤 숫자만 보고 cold start 문제를 무시함
- 너무 약한 부하를 걸어 hot code가 생기지 않게 측정함
- 클래스 로딩, 커넥션 초기화, 캐시 채우기를 JIT와 구분하지 않음
- 평균값만 보고 p95, p99 같은 꼬리 지연을 보지 않음
- 로컬 테스트 결과를 그대로 운영에 일반화함

## 실무에서 같이 보면 좋은 관측 포인트

- readiness까지 걸린 시간
- 첫 1분, 5분의 p95/p99 latency
- 초반 처리량 변화
- CPU 사용률과 compiler thread 사용 패턴
- 클래스 로딩 로그 또는 초기화 구간
- GC 로그와 allocation rate
- 오토스케일 이벤트 시점과 응답 시간 변화

## 대응 방법

### 1. 인스턴스를 미리 데우기

- readiness 이후 실제 사용자 트래픽을 받기 전에 synthetic traffic을 넣는다.
- 자주 쓰는 API, 쿼리, 직렬화 경로를 미리 한 번씩 태운다.
- 단순 헬스체크만으로는 충분하지 않은 경우가 많다.

### 2. 너무 빨리 죽지 않게 운영하기

- scale-to-zero나 과도한 pod churn을 줄인다.
- 최소 인스턴스를 유지해 완전히 차가운 시작을 줄인다.
- 인스턴스당 트래픽이 너무 낮아 hot path가 형성되지 않는 구조인지 확인한다.

### 3. startup 개선과 warmup 개선을 분리하기

startup 개선 예시:

- 의존성 축소
- 클래스 수 감소
- 불필요한 자동 설정 제거
- Class Data Sharing 사용 검토

warmup 개선 예시:

- pre-warming 트래픽
- 캐시 선적재
- 자주 호출되는 경로를 미리 실행
- steady-state 기준으로 pod 교체 정책 재검토

## CDS와 AppCDS가 왜 도움이 되나

`CDS(Class Data Sharing)`는 JVM이 자주 쓰는 클래스 메타데이터를 공유 아카이브로 써서 startup 시간과 메모리 사용량을 줄이는 기능이다. `AppCDS`는 이를 애플리케이션 클래스까지 확장한다.

- 클래스 로딩 부담 일부를 줄일 수 있다.
- 여러 JVM 프로세스가 같은 호스트에서 공유 이점을 볼 수 있다.
- 주로 `startup` 쪽 개선에 더 직접적이다.

즉, CDS는 warmup 전체를 없애는 만능 도구라기보다 `초기 준비 비용`을 줄이는 데 가깝다.

## 벤치마크는 JMH가 왜 자주 언급되나

일반적인 `System.nanoTime()` 측정은 JVM 최적화, dead code elimination, warmup 편차에 쉽게 속는다. `JMH`는 JVM 계열 마이크로벤치마크를 더 안전하게 만들기 위한 표준 도구다.

- warmup iteration과 measurement iteration을 분리할 수 있다.
- 벤치마크 함정에 덜 빠지게 돕는다.
- 작은 코드 조각 비교에는 유용하지만, 서비스 전체 startup 문제를 그대로 대변하지는 않는다.

## 어떤 선택이 맞는가

- 장수명 서버: steady state와 운영 안정성이 우선인 경우가 많다.
- 짧은 배치/워커: warmup 단축이 더 중요할 수 있다.
- 서버리스/오토스케일: cold start와 warmup을 제품 성능의 일부로 봐야 한다.

## 실전 체크리스트

1. readiness 시간과 첫 1분 p99 latency를 따로 측정했는가
2. 초반 느림의 원인을 JIT, 초기화, 캐시, GC 중 어디까지 분리했는가
3. 새 인스턴스가 실제 사용자 트래픽 전 pre-warming 되는가
4. scale policy가 startup CPU spike를 과민하게 해석하지 않는가
5. steady-state 수치만으로 의사결정하고 있지 않은가

## 한 줄 결론

JVM warmup 대응의 핵심은 `JVM을 더 세게 튜닝`하기 전에 `무엇이 startup 문제이고 무엇이 warmup 문제인지 먼저 분리해서 측정`하는 것이다.
