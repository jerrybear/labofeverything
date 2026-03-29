# MySQL Master-Slave Replication Lag 집중 가이드

## 이 문서는 무엇을 설명하나

이 문서는 MySQL master-slave replication lag를 운영 관점에서 집중적으로 설명합니다. 현재 공식 용어는 source-replica이지만, 현업에서 여전히 master-slave라는 표현을 많이 쓰므로 그 표현도 함께 사용합니다.

이 문서의 목적은 다음입니다.

- replication lag가 정확히 무엇인지 이해하기
- 왜 lag가 발생하는지 운영 관점으로 정리하기
- 어떤 상황에서 위험 신호로 봐야 하는지 판단하기
- incident가 났을 때 어떤 순서로 봐야 하는지 정리하기

## 먼저 한 줄로 요약하면

- replication lag는 보통 slave가 master의 변경을 늦게 따라가는 상태입니다.
- 실제 원인은 "slave 자체가 느리다"보다도, 큰 트랜잭션, 대량 DML, 비효율적인 apply, lock contention, 하드웨어 차이 때문에 생기는 경우가 많습니다.
- 운영에서는 lag 숫자 하나만 보는 것이 아니라, write shape, apply 병목, replica read workload, 사용자 영향까지 같이 봐야 합니다.

## 1. replication lag란 무엇인가

master에 이미 반영된 변경이 slave에는 아직 반영되지 않은 상태를 뜻합니다.

운영에서 이 상태가 위험한 이유:

- slave read가 stale read가 될 수 있음
- 읽기 분산 구조에서 사용자마다 다른 데이터를 볼 수 있음
- failover 시 최신 데이터 기준이 흔들릴 수 있음
- 운영자가 보고 있는 지표와 실제 최신 데이터가 어긋날 수 있음

즉, lag는 단순한 내부 지표가 아니라, 읽기 일관성과 장애 대응 신뢰도에 직접 영향을 주는 지표입니다.

## 2. 가장 흔한 원인

### 1) 큰 트랜잭션

가장 흔한 원인 중 하나입니다.

- master에서 큰 트랜잭션이 오래 실행됨
- commit 전까지는 slave가 완전히 따라갈 수 없음
- commit 후에는 slave가 큰 apply 작업을 한 번에 떠안음

운영에서는 "master는 멀쩡해 보였는데 commit 이후 slave lag가 갑자기 튄다"는 패턴으로 자주 보입니다.

### 2) 대량 DML

- bulk `INSERT`
- bulk `UPDATE`
- bulk `DELETE`

이런 작업은 slave apply throughput을 쉽게 압도할 수 있습니다.

특히 다음 상황이 위험합니다.

- chunk 없이 한 번에 큰 배치 실행
- 인덱스가 많아 쓰기 비용이 큰 테이블
- delete 후 purge 부담까지 큰 경우

### 3) apply 단계의 비효율

slave가 변경을 받아와도 적용이 느릴 수 있습니다.

- 인덱스 부족
- 비효율적인 lookup
- lock contention
- 느린 스토리지
- 낮은 CPU/메모리 자원

즉, network receive는 괜찮은데 apply가 병목인 경우가 운영에서 매우 흔합니다.

### 4) slave 위의 read workload

slave는 replication만 하는 장비가 아닌 경우가 많습니다.

- read traffic
- analytics query
- export job
- ad hoc query

이런 작업이 apply와 경쟁하면 lag가 쉽게 커집니다.

### 5) DDL과 유지보수 작업

- 인덱스 추가
- 컬럼 변경
- partition drop
- 기타 schema change

이런 작업은 master에서의 실행 비용뿐 아니라 slave의 재실행/적용 지연까지 함께 봐야 합니다.

## 3. 운영에서 보이는 대표 증상

- `Seconds_Behind_Master` 또는 유사 지표 증가
- slave read 결과가 오래된 상태로 보임
- 배치 완료 후에도 slave에서는 결과 반영이 늦음
- failover readiness가 떨어짐
- DDL 또는 bulk DML 직후 lag spike 발생

실무에서는 특히 아래 조합이 자주 보입니다.

- bulk `UPDATE` 후 lag 급증
- partition drop 또는 인덱스 작업 직후 lag 증가
- slave에서 긴 read query가 돌면서 apply 정체
- master는 여유로운데 slave만 늦어짐

## 4. 어떻게 분해해서 봐야 하나

lag는 크게 두 단계로 나눠 생각하는 것이 좋습니다.

### A. 받아오는 단계가 느린가

즉, master의 변경을 slave가 충분히 빨리 가져오지 못하는 경우입니다.

가능한 원인:

- 네트워크 문제
- source/master 연결 이슈
- I/O thread 문제

### B. 적용 단계가 느린가

즉, 받아온 변경은 있는데 slave가 충분히 빨리 반영하지 못하는 경우입니다.

가능한 원인:

- 너무 무거운 write workload
- 큰 트랜잭션
- apply 중 lock wait
- slave 리소스 부족
- local read와의 경쟁

운영에서는 B가 훨씬 더 흔합니다.

## 5. 먼저 무엇을 확인할까

### 기본 상태 확인

```sql
SHOW REPLICA STATUS\G
```

master-slave 표현을 쓰는 환경에서는 다음도 볼 수 있습니다.

```sql
SHOW SLAVE STATUS\G
```

여기서 보는 핵심:

- I/O thread 상태
- SQL/applier thread 상태
- 에러 존재 여부
- lag가 증가 중인지, 멈췄는지, 회복 중인지

### process list 확인

```sql
SHOW FULL PROCESSLIST;
```

봐야 할 것:

- slave에서 긴 read query가 있는지
- apply thread가 lock이나 I/O 때문에 막혔는지
- 오래 열린 transaction이 있는지

### InnoDB 상태 확인

```sql
SHOW ENGINE INNODB STATUS\G
```

봐야 할 것:

- lock wait
- 긴 transaction
- purge/history list 증가
- 백그라운드 압박

## 6. 원인별 운영 해석

### 큰 트랜잭션이 원인일 때

징후:

- 평소엔 괜찮다가 commit 이후 lag 급등
- bulk batch 직후 slave가 오래 따라잡지 못함

운영 해석:

- 트랜잭션 크기를 줄이는 것이 1순위 해결책
- 한 번에 몰아치는 배치보다 chunk가 안전함

### 대량 DML이 원인일 때

징후:

- source의 rows changed가 급증
- 동시에 slave lag도 증가
- replica apply throughput이 못 따라감

운영 해석:

- source write shape를 먼저 줄여야 함
- slave만 만져서는 근본 해결이 안 되는 경우가 많음

### slave read workload가 원인일 때

징후:

- slave 위에서 긴 조회 쿼리가 발견됨
- apply thread가 lock wait 또는 리소스 경쟁 상태

운영 해석:

- incident 시에는 slave의 "읽기 편의성"보다 apply 건강을 우선해야 함

## 7. 어느 정도부터 위험 신호인가

정답은 시스템마다 다르지만, 운영에서는 보통 이렇게 봅니다.

- 잠깐 튀었다가 금방 회복: 관찰 수준
- 수십 초 이상 지속: 경고 수준
- 수분 이상 지속: 운영 개입 필요
- stale read가 실제 사용자에게 보임: 이미 incident 수준

중요한 것은 절대값보다:

- 증가 속도
- 지속 시간
- 사용자 영향
- failover 준비도

입니다.

## 8. 운영 대응 패턴

### 1) batch가 원인이라면

- batch size를 줄임
- chunk 사이 pause를 둠
- lag threshold를 넘으면 중단함

### 2) slave read가 원인이라면

- 긴 query를 식별함
- 중요하지 않은 analytic/export 쿼리를 멈춤
- replica apply를 우선시함

### 3) slave가 너무 약하다면

- source와 slave 하드웨어 차이를 봄
- CPU, 메모리, storage throughput을 비교함
- 더 무거운 source를 더 작은 slave가 따라갈 것이라고 가정하지 않음

### 4) DDL이 원인이라면

- 인덱스/컬럼/PK 변경 작업 여부를 확인함
- 같은 시점의 metadata lock 또는 rebuild성 작업을 의심함
- replica 재실행 비용을 함께 고려함

## 9. 예방 전략

### 트랜잭션을 작게 유지하기

가장 효과적인 예방책 중 하나입니다.

### 대량 작업은 backpressure를 넣기

- chunk 처리
- pause
- lag threshold 기반 중단

### slave를 너무 바쁘게 쓰지 않기

- 읽기 전용 서비스와 분석 작업을 무분별하게 몰지 않기
- 긴 query를 상시 돌리는 구조 피하기

### DDL은 replication까지 포함해 계획하기

- master만 보고 끝내지 않기
- slave apply 영향과 lag 회복 시간을 같이 계산하기

## 10. 빠른 체크리스트

lag가 보이면:

1. `SHOW REPLICA STATUS\G` 또는 `SHOW SLAVE STATUS\G` 확인
2. receive 병목인지 apply 병목인지 구분
3. 최근 bulk DML/DDL 유입 확인
4. slave의 긴 read query와 lock wait 확인
5. batch 또는 트래픽을 줄일지 판단
6. lag가 실제로 줄기 시작하는지 확인

## 11. 같이 보면 좋은 문서

- `replication-lag-troubleshooting.md`
- `bulk-dml-runbook.md`
- `ddl-and-dml-interaction.md`
- `transaction-history-metric.md`

## 한 줄 결론

- master-slave replication lag는 slave가 느린 문제가 아니라, master의 write shape와 slave의 apply 한계가 부딪히는 운영 문제로 보는 것이 가장 정확합니다.

## 참고 자료

- MySQL Reference Manual, InnoDB and Replication
- MySQL Reference Manual, Replication Status and Administration
- MySQL Reference Manual, Replication Problems and Troubleshooting
