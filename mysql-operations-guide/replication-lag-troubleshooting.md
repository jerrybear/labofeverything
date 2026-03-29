# MySQL Replication Lag 트러블슈팅 가이드

## 이 가이드는 누구를 위한 문서인가

이 문서는 MySQL replica가 왜 뒤처지는지 이해하고, 상황을 더 악화시키지 않으면서 안전하게 대응해야 하는 운영자와 엔지니어를 위한 가이드입니다.

## 핵심 요약

- replication lag는 replica 자체 문제라기보다 write shape 문제인 경우가 많습니다.
- 하나의 큰 트랜잭션만으로도 source가 멀쩡해 보이는 동안 갑작스러운 lag spike가 생길 수 있습니다.
- lag는 네트워크 수신 지연, relay log 누적, apply thread 속도 저하, replica의 lock contention, 단순한 하드웨어 한계 등 다양한 원인으로 생깁니다.
- 가장 안전한 첫 단계는 replica가 변경을 못 받아오는 것인지, 받아온 변경을 못 적용하는 것인지 구분하는 것입니다.

## 1. 운영 관점에서 replication lag란 무엇인가

운영 관점에서 lag는 source에 이미 커밋된 데이터가 replica에서는 아직 동일하게 보이지 않는 상태를 뜻합니다.

이 상태는 다음과 같은 실제 위험을 만듭니다.

- stale read
- 운영 대시보드 왜곡
- failover 준비도 저하
- 읽기 트래픽이 replica로 갈 때 사용자에게 보이는 데이터 불일치

## 2. 가장 흔한 원인

### 큰 트랜잭션

가장 흔한 원인 중 하나는 source에서 발생한 매우 큰 트랜잭션입니다.

- 트랜잭션은 commit되기 전까지 replication에 완전히 드러나지 않음
- commit 후 replica는 갑자기 큰 양의 apply 작업을 떠안게 됨
- 그래서 "갑자기 lag가 생겼다"처럼 보일 수 있음

### DML-heavy 워크로드

bulk `UPDATE`, `DELETE`, `INSERT`는 replica apply throughput을 쉽게 압도할 수 있습니다.

### 인덱스 부족 또는 비효율적인 키 구조

복제된 row change를 적용하는 과정에서 lookup이 비효율적이거나 scan이 커지면 apply 작업이 느려집니다.

### replica의 lock contention

replica도 lock wait와 deadlock을 겪을 수 있습니다. 특히 다음 경우에 그렇습니다.

- 읽기 트래픽이 replica로도 들어올 때
- 긴 조회 쿼리가 replica에서 돌고 있을 때
- apply worker가 local workload와 경쟁할 때

### 과소한 replica 하드웨어

replica의 CPU, 메모리, 스토리지 처리량이 source보다 낮으면 bursty write에서 lag가 더 쉽게 발생합니다.

### DDL과 유지보수 작업

schema 변경과 유지보수 작업은 apply를 막거나 느리게 만들 수 있습니다.

## 3. lag는 어떻게 드러나는가

운영 환경에서 흔히 보이는 신호:

- read-only endpoint가 오래된 결과를 반환함
- 모니터링에서 `Seconds_Behind_Source`가 증가함
- replica에 relay log가 쌓임
- replica가 최신 상태가 아니어서 failover 신뢰도가 떨어짐
- source에서는 배치가 끝났는데 downstream read는 오래된 상태로 남음

중요한 주의점:

- 하나의 lag metric은 유용하지만 전부는 아님
- 수신 지연과 적용 지연을 구분해서 봐야 함

## 4. 먼저 무엇을 봐야 하나

### Replica 상태

```sql
SHOW REPLICA STATUS\G
```

다음을 확인합니다.

- I/O thread가 정상인지
- SQL 또는 applier thread가 정상인지
- 오류가 있는지
- lag가 증가 중인지, 정체 상태인지, 줄어드는 중인지

### Replica의 process list

```sql
SHOW FULL PROCESSLIST;
```

다음을 확인합니다.

- 오래 걸리는 read query
- 대기 중인 트랜잭션
- lock 또는 I/O 때문에 막힌 apply thread

### InnoDB 상태

```sql
SHOW ENGINE INNODB STATUS\G
```

다음을 확인합니다.

- lock wait
- 긴 트랜잭션
- history list 증가
- 무거운 백그라운드 압박의 징후

### Source 쪽 workload shape

source에서 최근에 아래 작업이 있었는지 확인합니다.

- bulk update 또는 delete
- 큰 insert burst
- schema change
- 최근에 커밋된 긴 트랜잭션

지금 source가 멀쩡해 보여도 replica에는 이미 큰 apply backlog를 남겼을 수 있습니다.

## 5. 문제를 어떻게 분해해서 볼 것인가

lag는 크게 두 부류로 나눌 수 있습니다.

### 변경을 충분히 빨리 받아오지 못하는 경우

가능한 원인:

- 네트워크 문제
- source 연결 문제
- replication thread 문제

### 변경은 받아오지만 적용을 충분히 빨리 못 하는 경우

가능한 원인:

- 쓰기 workload가 너무 무거움
- replica의 lock 문제
- 너무 큰 트랜잭션
- 스토리지 병목
- replica compute resource 부족

이 구분이 있으면 운영자는 추측 대신 진단할 수 있습니다.

## 6. 안전한 대응 패턴

### bulk job이 lag를 만드는 경우

- job의 batch size를 줄임
- chunk 사이에 pause를 넣음
- lag가 합의한 임계값을 넘으면 작업을 멈춤
- 더 작은 트랜잭션으로 다시 시작함

### local read가 replica를 막고 있는 경우

- replica 위의 긴 read query를 식별함
- 중요하지 않은 analytic query를 이동하거나 일시 중단함
- incident 중에는 편의성 트래픽보다 replica apply 건강을 우선함

### replica가 단순히 너무 작은 경우

- source와 replica 하드웨어 크기를 비교함
- 필요한 CPU, 메모리, 스토리지 처리량을 늘림
- 더 무거운 source를 더 작은 replica가 따라갈 것이라고 가정하지 않음

### 오류가 존재하는 경우

- replication error는 성능 문제 이전에 correctness 문제로 다룸
- 운영상 위험을 명확히 이해하기 전에는 함부로 transaction skip을 하지 않음

## 7. 계속 봐야 할 모니터링 신호

최소한 다음은 모니터링해야 합니다.

- 시간에 따른 lag 추이
- source write throughput
- replica apply throughput
- relay log 증가량
- replica CPU와 storage latency
- replica의 lock wait와 deadlock
- replica 위의 long-running query

유용한 명령과 view:

```sql
SHOW REPLICA STATUS\G
SHOW FULL PROCESSLIST;
SHOW ENGINE INNODB STATUS\G
```

활성화돼 있다면 `performance_schema`와 `sys`는 wait event와 바쁜 statement를 더 체계적으로 추적하는 데 도움이 됩니다.

## 8. 안전 임계값과 stop condition

각 시스템은 자신만의 lag 허용치를 정해야 하지만, 운영자는 명확한 incident threshold를 사전에 합의해야 합니다.

예시:

- lag가 몇 분 동안 30초를 넘으면 경고
- lag가 60초를 넘으면 bulk job 일시 중지
- stale read 위험이 허용 범위를 넘으면 replica-directed user traffic 중단

중요한 것은 정확한 숫자 자체보다, 사전에 합의된 대응이 있다는 점입니다.

## 9. 예방 패턴

### 트랜잭션을 더 작게 유지하기

작은 트랜잭션은 replica가 흡수하기 더 쉽습니다.

### 모든 테이블에 적절한 키를 갖추기

primary key와 좋은 인덱스는 correctness뿐 아니라 apply 효율에도 중요합니다.

### replica workload를 조심하기

replica가 읽기 트래픽, analytics, export, replication을 동시에 감당하면 bursty write 아래에서 쉽게 무너질 수 있습니다.

### backpressure가 있는 batching 사용하기

유지보수 작업은 lag에 따라 속도를 늦추거나 pause할 수 있어야 합니다.

### parallel apply 설정 검토하기

replica parallelism은 도움이 될 수 있지만 만능은 아닙니다. workload shape가 나쁘면 worker를 늘릴수록 lock competition이 더 심해질 수도 있습니다.

## 10. 추천 트러블슈팅 흐름

1. lag가 실제로 존재하고 증가 중인지 확인함
2. receive가 병목인지 apply가 병목인지 구분함
3. 최근 source-side write shape를 확인함
4. replica의 lock wait, 긴 read, 하드웨어 압박을 점검함
5. 필요하면 write generator를 줄이거나 멈춤
6. lag 추세가 실제로 줄기 시작했는지 확인한 뒤 복구 선언을 함

## 11. 빠른 체크리스트

개입 전:

- lag metric 확인
- source와 replica 상태 확인
- 최근 bulk 작업이나 schema 작업 확인

개입 중:

- lag 추세 관찰
- apply thread 동작 관찰
- replica lock과 긴 read 관찰

개입 후:

- replica가 따라잡았는지 확인
- 숨은 backlog가 남지 않았는지 확인
- 다음을 위한 안전한 batch size와 lag threshold 기록

## 참고 자료

- MySQL Reference Manual, Replication Status and Administration
- MySQL Reference Manual, Replication Problems and Troubleshooting
- MySQL Reference Manual, InnoDB and Replication
