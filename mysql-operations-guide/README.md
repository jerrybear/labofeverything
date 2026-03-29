# MySQL 운영 가이드

## 이 가이드는 누구를 위한 문서인가

이 문서는 운영자, 백엔드 엔지니어, 개발자가 MySQL을 운영 관점에서 이해할 수 있도록 만든 가이드입니다. 특히 `INSERT`, `UPDATE`, `DELETE` 같은 DML 때문에 운영 환경에서 어떤 일이 벌어지는지에 초점을 맞춥니다.

## 핵심 요약

- MySQL에서 DML 부하는 보통 "한 줄이 바뀌었다"로 끝나지 않습니다. 메모리 변경, redo/undo 로그 기록, 인덱스 유지, 락, flush, replication 작업이 함께 일어납니다.
- 가장 먼저 보이는 증상은 CPU 포화보다 지연 시간 급등, lock wait, 디스크 압박, replica lag인 경우가 많습니다.
- `UPDATE`와 `DELETE`는 이전 버전 관리, 보조 인덱스, purge, 긴 트랜잭션과 얽히기 때문에 `INSERT`보다 운영상 더 아픈 경우가 많습니다.
- 큰 트랜잭션은 위험합니다. rollback 비용, 락 유지 시간, undo 증가, 장애 복구 시간, replication 지연을 모두 키웁니다.

## 1. 운영 관점에서 중요한 MySQL 구성 요소

운영 관점에서는 다음 구성 요소가 특히 중요합니다.

- `InnoDB buffer pool`: 데이터와 인덱스 페이지를 메모리에 캐시합니다.
- `redo log`: 장애 복구를 위해 커밋된 변경 사항을 기록합니다.
- `undo log`: rollback과 MVCC 읽기를 위해 이전 버전의 행을 유지합니다.
- `change buffer`: 일부 보조 인덱스 작업을 늦춰 즉시 발생하는 I/O를 줄입니다.
- `binary log`: replication과 복구 워크플로를 위해 변경 사항을 기록합니다.
- `locks and transactions`: 동시성, 대기, rollback 범위를 결정합니다.

DML 부하가 커지면 이 요소들이 동시에 활발하게 움직이기 시작합니다.

## 2. DML이 시스템 부하를 만드는 이유

### `INSERT`

`INSERT`는 단순해 보여도 MySQL은 다음 작업을 수행할 수 있습니다.

- 메모리에서 데이터 페이지를 할당하고 수정함
- redo log 레코드를 기록함
- 트랜잭션 처리에 필요하면 undo 정보를 기록함
- 기본 키와 보조 인덱스를 갱신함
- 나중에 dirty page를 디스크로 flush함
- replication 또는 시점 복구가 활성화돼 있으면 binary log 이벤트를 기록함

운영 관점에서 이는 대량 insert가 로그 쓰기 압박, 인덱스 유지 비용, 이후 flush 압박의 조합으로 이어질 수 있다는 뜻입니다.

### `UPDATE`

`UPDATE`는 생각보다 더 무거운 경우가 많습니다. MySQL이 다음 작업을 해야 하기 때문입니다.

- 먼저 대상 행을 찾아야 함
- 일치하는 행 또는 범위에 락을 걸어야 함
- rollback과 MVCC 가시성을 위해 이전 버전을 보존해야 함
- clustered record를 변경해야 함
- 인덱싱된 컬럼이 바뀌면 보조 인덱스도 갱신해야 함
- redo log와 binary log를 기록해야 함

조건이 넓거나 특정 행이 hot row라면 `UPDATE`는 아주 빠르게 lock contention과 latency spike를 만들 수 있습니다.

### `DELETE`

대량 `DELETE`는 운영상 특히 위험합니다.

- 행이 항상 즉시 제거되는 것이 아니라 표시되고 버전이 관리됨
- statement가 끝난 뒤에도 undo와 purge 작업이 계속될 수 있음
- 보조 인덱스 엔트리도 함께 처리해야 함
- replica 역시 같은 delete 작업을 적용해야 함

그 결과, 하나의 큰 delete가 쿼리 실행 시간보다 훨씬 오래 foreground 트래픽에 악영향을 줄 수 있습니다.

## 3. 부하는 어디에서 드러나는가

### 디스크 I/O

DML이 많은 시스템에서 가장 흔한 병목은 스토리지 I/O입니다.

- durability 설정에 따라 commit 시점에 redo log flush가 일어남
- dirty page는 결국 buffer pool에서 디스크로 flush되어야 함
- checkpoint 압박이 write burst를 키울 수 있음
- 보조 인덱스가 많거나 단편화가 심하면 random I/O가 증가함

### 메모리 압박

buffer pool이 너무 작으면 다음 현상이 나타납니다.

- 유용한 페이지가 너무 자주 축출됨
- 다시 페이지를 읽어와야 하는 읽기 작업이 늘어남
- dirty page 관리가 더 공격적으로 일어남
- flush 작업이 foreground 작업을 방해할 수 있음

### CPU 비용

CPU가 병목이 되는 경우도 있습니다.

- 작은 트랜잭션이 매우 많이 동시 실행될 때
- 보조 인덱스 유지 작업이 많을 때
- 비효율적인 조건으로 너무 많은 행을 스캔할 때
- contention 때문에 락 관리와 재시도 비용이 늘어날 때

### 락과 동시성

DML은 다음 문제를 유발할 수 있습니다.

- row lock wait
- gap 또는 next-key lock contention
- deadlock
- 긴 트랜잭션으로 인한 blocking chain

운영 환경에서는 핵심 원인이 일부 락 충돌인데도 사용자들은 보통 "DB가 갑자기 느려졌다"고 느끼게 됩니다.

## 4. 운영자가 알아야 할 InnoDB 세부 사항

### Buffer pool

buffer pool은 InnoDB의 핵심 작업 메모리 영역입니다. 활성 데이터와 인덱스가 메모리에 잘 올라가면 MySQL 성능이 훨씬 좋아집니다.

운영 팁:

- 전용 DB 호스트에서는 `innodb_buffer_pool_size`를 RAM의 대략 70~80% 수준으로 잡는 경우가 많음
- 캐시 효율 저하와 과도한 flush를 관찰해야 함
- pool이 커도 큰 쓰기 burst는 dirty-page 압박을 만들 수 있음

### Redo log

redo log는 데이터 파일이 완전히 갱신되기 전에 커밋된 변경 사항을 보호합니다.

운영 팁:

- commit가 많은 워크로드는 redo flush를 강하게 압박함
- durability 설정은 성능과 안정성의 trade-off를 만듦
- redo 용량이 작으면 checkpoint가 자주 발생해 write 압박이 커질 수 있음

### Undo log와 purge

undo는 rollback과 MVCC를 지원합니다. 이전 버전의 행은 purge가 정리할 때까지 남아 있습니다.

운영 팁:

- 긴 트랜잭션은 purge를 지연시킴
- 큰 update와 delete는 history length를 키울 수 있음
- purge 지연은 백그라운드 작업과 스토리지 사용량을 부풀릴 수 있음

### Change buffer

일부 보조 인덱스 작업에서는 InnoDB가 즉시 페이지를 읽는 대신 변경을 나중에 병합할 수 있습니다.

운영 팁:

- 일부 워크로드에서는 즉시 발생하는 I/O를 줄일 수 있음
- 일을 없애는 것이 아니라 일부 비용을 나중으로 미루는 것임
- 백그라운드 merge 비용이 나중에 나타날 수 있다는 점을 운영자가 인지해야 함

## 5. 운영 환경에서 흔한 증상

DML 부하가 건강하지 않은 상태가 되면 흔히 다음 증상이 보입니다.

- 쓰기 피크 때 애플리케이션 응답 시간이 상승함
- lock wait timeout이나 deadlock이 늘어남
- `SHOW PROCESSLIST`에 lock 또는 commit 대기 세션이 많이 보임
- 스토리지 latency 또는 `iowait`가 높아짐
- bulk change 이후 replica lag가 증가함
- 매우 큰 트랜잭션이 살아 있던 상태에서 비정상 종료되면 복구가 느려짐

증상과 가능한 원인을 연결하면 대체로 다음과 같습니다.

| 증상 | 가능한 원인 |
| --- | --- |
| commit latency 급등 | redo flush 압박, 스토리지 latency, 너무 많은 동시 commit |
| 특정 테이블만 갑자기 느려짐 | hot row, 인덱스 부족, lock contention |
| replica lag | primary write burst, 큰 트랜잭션, replica apply 지연 |
| CPU는 보통인데 디스크가 바쁨 | flush/checkpoint 압박, 인덱스 write amplification |
| rollback이 오래 걸림 | 매우 큰 트랜잭션, 긴 undo chain |

## 6. 인덱스가 쓰기를 더 무겁게 만드는 이유

인덱스는 읽기를 돕지만, 모든 쓰기 작업은 인덱스도 함께 만질 수 있습니다.

- 각 보조 인덱스는 추가 페이지 갱신 비용을 만듦
- 인덱싱된 컬럼을 바꾸는 일은 비인덱스 컬럼보다 훨씬 비쌀 수 있음
- 인덱스가 너무 많으면 write amplification이 증가함

운영 관점의 기본 원칙:

- 쓰기 비중이 높은 테이블에서는 새 인덱스 하나도 강한 근거가 있어야 함

## 7. Replication 영향

DML 부하는 primary에서 끝나지 않습니다.

- 변경 사항은 binary log에도 기록됨
- replica는 같은 작업을 받아서 다시 적용해야 함
- 하나의 거대한 트랜잭션은 replica가 따라잡기 어렵게 만듦
- apply 지연은 replica lag와 stale read로 나타남

운영상 의미:

- primary에서는 괜찮아 보이는 bulk update도 replica에는 과부하를 줄 수 있음
- 하나의 큰 트랜잭션보다 작은 배치가 보통 더 안전함
- correctness가 중요한 워크로드에서는 row-based replication이 운영상 더 안전한 경우가 많음

## 8. DML-heavy 시스템의 안전한 운영 방식

### 배치를 선호하기

거대한 한 문장보다 아래 방식이 더 안전합니다.

- multi-row `INSERT`로 insert를 배치 처리함
- update와 delete를 chunk로 나눔
- 큰 트랜잭션 하나를 오래 잡지 말고 중간중간 commit함

예시 패턴:

```sql
DELETE FROM orders
WHERE created_at < '2024-01-01'
LIMIT 1000;
```

수백만 건을 한 번에 지우기보다, pause나 제어된 스케줄링을 두고 반복 실행하는 편이 안전합니다.

### 긴 트랜잭션 피하기

긴 트랜잭션은 다음 문제를 만듭니다.

- 락을 더 오래 잡음
- purge를 지연시킴
- rollback 고통을 키움
- replica가 따라잡기 어렵게 만듦

### 점검 창을 신중하게 선택하기

큰 데이터 변경은 다음 요소를 고려해 스케줄링해야 합니다.

- 피크 트래픽 시간대
- replica 상태
- 백업 시간대
- 스토리지 처리량 한계

### 실제 규모로 테스트하기

작은 데이터셋에서 괜찮아 보이는 쿼리라도, 운영 데이터 규모와 동시성에서는 완전히 다르게 동작할 수 있습니다.

## 9. 무엇을 모니터링해야 하는가

운영자는 사용자 관점의 신호와 DB 내부 신호를 함께 봐야 합니다.

### 사용자 관점 신호

- API latency
- error rate
- timeout rate
- 의존 시스템의 queue 증가

### 데이터베이스 신호

- transaction throughput
- commit latency
- lock wait와 deadlock
- 초당 변경된 행 수
- buffer pool hit 동작
- dirty page 압박
- redo log 압박
- replication lag
- storage latency와 IOPS

### 유용한 내장 명령

```sql
SHOW ENGINE INNODB STATUS\G
SHOW FULL PROCESSLIST;
SHOW GLOBAL STATUS LIKE 'Innodb%';
SHOW GLOBAL STATUS LIKE 'Threads%';
```

### 유용한 schema view

활성화돼 있다면 `performance_schema`와 `sys`는 다음 질문에 답하는 데 도움이 됩니다.

- 어떤 statement가 가장 자주 느린가
- 대기는 어디에서 발생하는가
- 어떤 테이블이 가장 뜨거운가
- replication worker가 뒤처지고 있는가

## 10. 실전 대응 플레이북

### 쓰기 latency가 갑자기 튀는 경우

- 먼저 스토리지 latency와 commit 동작을 확인함
- active transaction과 lock wait를 점검함
- write 패턴을 바꾼 새 배치 작업이나 배포가 있었는지 확인함
- replica도 함께 뒤처지고 있는지 확인함

### lock wait가 증가하는 경우

- blocking session을 식별함
- hot table과 hot row를 찾음
- 트랜잭션이 더 넓어졌거나 더 길어졌는지 확인함
- update와 delete가 불필요한 scan을 하지 않도록 인덱스 사용을 검증함

### replica lag가 증가하는 경우

- 최근에 하나의 큰 트랜잭션이 커밋됐는지 확인함
- primary write throughput과 replica apply throughput을 비교함
- 유지보수 작업의 batch size를 줄임
- replica 하드웨어가 과소한지 검토함

### 큰 delete가 필요한 경우

- chunk 단위로 지움
- 실행 중 lag와 lock wait를 함께 모니터링함
- 필요하면 chunk 사이에 pause를 둠
- 시작 전에 stop condition과 rollback 전략을 정해둠

## 11. 이해를 돕는 간단한 사고 모델

운영 관점에서는 DML을 아래처럼 이해하면 좋습니다.

1. MySQL이 메모리 페이지를 변경한다.
2. MySQL이 로그에 안전 정보를 기록한다.
3. MySQL이 인덱스와 트랜잭션 메타데이터를 갱신한다.
4. MySQL이 락과 MVCC로 동시성을 조정한다.
5. MySQL이 결국 디스크와 replication 비용을 지불한다.

그래서 write-heavy 워크로드는 처음에는 멀쩡해 보이다가도, 나중에 flush 압박, purge 부채, replica lag로 인해 급격히 나빠질 수 있습니다.

## 12. 운영자 체크리스트

큰 쓰기 작업 전에:

- 대상 row 수를 확인함
- `EXPLAIN`으로 인덱스 사용을 확인함
- chunk size를 정함
- 트래픽 창이 안전한지 확인함
- replica 상태를 확인함
- rollback과 stop 전략을 확인함

실행 중:

- latency를 봄
- lock wait와 deadlock을 봄
- 스토리지와 replica lag를 봄
- 사용자 영향이 임계값을 넘으면 중단함

실행 후:

- replica가 회복됐는지 확인함
- 긴 트랜잭션이 남지 않았는지 확인함
- purge나 백그라운드 작업이 여전히 높은지 확인함
- 다음을 위해 안전한 batch size를 기록함

## 13. 다음에 이어서 만들기 좋은 주제

이 가이드가 유용하다면 다음 문서도 자연스럽게 이어질 수 있습니다.

1. MySQL lock과 deadlock 가이드
2. MySQL replication lag troubleshooting 가이드
3. 안전한 bulk update/delete runbook
4. MySQL 모니터링 대시보드 체크리스트

## 함께 보는 문서

- `locks-and-deadlocks.md`
- `replication-lag-troubleshooting.md`
- `master-slave-replication-lag-guide.md`
- `bulk-dml-runbook.md`
- `transaction-history-metric.md`
- `ddl-and-dml-interaction.md`
- `native-online-ddl-vs-ptosc.md`
- `ddl-risk-summary.md`
- `metadata-lock-ddl-runbook.md`
- `index-drop-checklist.md`
- `index-drop-candidate-selection.md`
- `index-drop-post-checks.md`

## 참고 자료

- MySQL 8.0 Reference Manual, InnoDB optimization and transaction management
- MySQL 8.0 Reference Manual, InnoDB buffer pool
- MySQL 8.0 Reference Manual, InnoDB redo log
- MySQL 8.0 Reference Manual, change buffering
- MySQL 8.0 Reference Manual, Performance Schema and sys schema
- MySQL 8.0 Reference Manual, InnoDB and replication
