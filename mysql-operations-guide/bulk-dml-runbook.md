# MySQL 대량 DML 런북

## 이 가이드는 누구를 위한 문서인가

이 런북은 운영 환경에서 큰 `UPDATE` 또는 `DELETE` 작업을 돌려야 하지만, 불필요한 장애는 피하고 싶은 운영자와 엔지니어를 위한 문서입니다.

## 핵심 요약

- 가장 안전한 bulk DML 전략은 보통 작은 chunk, 잦은 commit, 명확한 stop condition입니다.
- 주요 위험은 lock spread, undo 증가, 긴 rollback 시간, replica lag, 사용자 지연 시간 급등입니다.
- 거대한 단일 트랜잭션은 운영 환경에서 대체로 잘못된 선택입니다.
- bulk job은 사전 점검, 실행 중 모니터링, 중단 계획을 갖춘 통제된 작업으로 다뤄야 합니다.

## 1. 시작 전에 해야 할 일

### 정확히 무엇이 바뀌는지 확인하기

같은 predicate를 사용하는 preview query를 먼저 실행합니다.

```sql
SELECT id
FROM orders
WHERE status = 'EXPIRED'
  AND created_at < '2024-01-01'
ORDER BY id
LIMIT 100;
```

그다음 대상 집합의 크기를 셉니다.

```sql
SELECT COUNT(*)
FROM orders
WHERE status = 'EXPIRED'
  AND created_at < '2024-01-01';
```

### 실행 계획 확인하기

```sql
EXPLAIN SELECT id
FROM orders
WHERE status = 'EXPIRED'
  AND created_at < '2024-01-01';
```

predicate가 너무 넓게 스캔된다면, lock risk와 runtime risk를 이해하기 전에는 진행하지 말아야 합니다.

### 운영 안전성 확인하기

실행 전에 다음을 확인합니다.

- 점검 창 또는 저트래픽 시간대인지
- replica 상태가 좋은지
- 최근 백업 또는 복구 준비 상태가 충분한지
- 작업을 pause하거나 stop하는 방법을 알고 있는지

## 2. 권장 실행 패턴

### chunk 사용하기

가장 일반적으로 안전한 패턴은 primary key 또는 다른 안정적인 인덱스 컬럼을 기준으로 작업을 chunk로 나누는 것입니다.

delete 예시:

```sql
DELETE FROM orders
WHERE id BETWEEN 100001 AND 101000
  AND status = 'EXPIRED';
```

update 예시:

```sql
UPDATE orders
SET archived = 1
WHERE id BETWEEN 100001 AND 101000
  AND status = 'COMPLETED';
```

### 자주 commit하기

각 chunk는 보통 하나의 독립된 트랜잭션이 되는 편이 좋습니다.

이유:

- lock duration이 짧아짐
- rollback 범위가 작아짐
- undo 축적이 줄어듦
- pause와 resume 지점을 만들기 쉬움

### 보수적으로 시작하기

처음에는 필요한 것보다 더 작은 chunk size로 시작하는 편이 안전합니다.

좋은 운영 태도:

- 먼저 안전성을 증명함
- 그다음 chunk size를 키움

## 3. chunk size 고르기

chunk size는 다음 요소에 따라 달라집니다.

- 테이블의 hotness
- 인덱스 품질
- row 크기
- replica 민감도
- 가용 maintenance window

실무에서의 시작점 예시:

- hot production table: 500 ~ 2,000 rows
- moderate traffic table: 2,000 ~ 10,000 rows
- cold maintenance table: 더 큰 chunk도 가능할 수 있음

정답은 하나가 아니라, 사용자 영향과 replica lag를 허용 범위 안에 두는 chunk size가 정답입니다.

## 4. 실행 중 주의해야 할 위험

### lock 위험

큰 statement나 인덱스가 좋지 않은 statement는 예상보다 훨씬 많은 row에 락을 걸 수 있습니다.

경고 신호:

- lock wait timeout 증가
- waiting session 증가
- foreground request 지연 증가

### rollback 위험

거대한 트랜잭션의 rollback은 원래 작업보다 훨씬 오래 걸릴 수 있습니다.

그래서 작은 단위로 commit된 chunk가 단일 거대 작업보다 훨씬 안전합니다.

### replica lag 위험

replica는 같은 변경 workload를 그대로 적용해야 합니다.

lag가 너무 빨리 커지면 source 쪽 bulk job이 너무 공격적이라는 뜻일 가능성이 큽니다.

### storage와 flush 압박

무거운 write는 한동안 괜찮아 보이다가도, dirty page, checkpoint, log flush 압박이 뒤늦게 따라오면서 급격히 나빠질 수 있습니다.

## 5. 작업이 도는 동안 무엇을 모니터링할까

### 애플리케이션 건강 상태

- API latency
- timeout rate
- error rate

### 데이터베이스 건강 상태

- active transaction
- lock wait와 deadlock
- chunk별 statement runtime
- storage latency
- replica lag

유용한 명령:

```sql
SHOW FULL PROCESSLIST;
SHOW ENGINE INNODB STATUS\G
SHOW REPLICA STATUS\G
```

### 한 번의 스냅샷보다 추세를 보기

좋은 1분이 안전을 증명해주지는 않습니다. 각 chunk 사이에서 상태가 안정적인지, 악화되는지, 회복되는지 추세를 봐야 합니다.

## 6. stop condition

작업 시작 전에 반드시 정해야 합니다.

예시:

- user-facing latency가 합의한 임계값을 초과함
- replica lag가 합의한 임계값을 초과함
- 같은 chunk 패턴에서 deadlock이 반복됨
- lock wait가 정상 트래픽에 영향을 주기 시작함
- chunk runtime이 계획한 안전 구간을 넘어감

stop condition을 넘으면:

- 새 chunk 투입을 멈춤
- 안전하다면 이미 실행 중인 작업만 마무리함
- chunk size와 트래픽 조건을 재평가함

## 7. 실전 운영 루프

각 chunk마다:

1. chunk 하나를 실행함
2. latency, lock, replica lag를 측정함
3. 필요하면 잠깐 쉼
4. 시스템이 건강할 때만 다음 chunk로 진행함

이런 backpressure 루프가 보통 "무조건 빨리 끝내기"보다 훨씬 안전합니다.

## 8. 대량 delete를 위한 안전 패턴

큰 delete 작업에서는:

- chunk delete를 선호함
- 수백만 건을 한 statement로 지우지 않음
- replica나 storage가 회복할 시간을 주기 위해 chunk 사이 pause를 둠

예시:

```sql
DELETE FROM audit_logs
WHERE created_at < '2024-01-01'
LIMIT 1000;
```

완료될 때까지 반복할 수는 있지만, 모니터링 없이 맹목적으로 루프를 돌리면 안 됩니다.

## 9. 대량 update를 위한 안전 패턴

큰 update 작업에서는:

- 가능하면 안정적인 ordering을 사용함
- predicate가 인덱스를 타도록 유지함
- 인덱싱된 컬럼은 꼭 필요할 때만 수정함. 쓰기 비용이 더 커지기 때문임
- 서로 다른 row에 서로 다른 값을 넣어야 하면, 많은 ad hoc statement 대신 staging-table 기반 update를 검토함

## 10. 작업이 끝난 뒤 확인할 것

다음을 확인합니다.

- 대상 row 수가 예상과 맞는지
- replica가 따라잡았는지
- 긴 트랜잭션이 남지 않았는지
- 남아 있는 lock 문제가 없는지
- application latency가 정상으로 돌아왔는지

그다음 다음 내용을 기록합니다.

- 최종 chunk size
- 총 수행 시간
- 가장 높았던 lag
- sleep 또는 pause가 필요했는지
- 다음에 더 안전하게 쓸 설정이나 절차

## 11. 아직 진행하면 안 되는 경우

다음 상황이라면 bulk job을 아직 돌리면 안 됩니다.

- 계획이 넓은 비인덱스 scan을 요구함
- replica가 이미 뒤처져 있음
- 시스템이 피크 트래픽 시간대임
- 명확한 pause 및 rollback 전략이 없음
- 정확한 대상 집합을 preview하지 않았음

## 12. 최소 체크리스트

시작 전:

- 대상 row preview
- row 수 count
- index path 검증
- chunk size 정의
- stop condition 정의

실행 중:

- latency 관찰
- lock 관찰
- lag 관찰
- 필요하면 chunk size 조정

실행 후:

- 데이터 결과 검증
- replica catch-up 검증
- 안전했던 절차 기록

## 참고 자료

- MySQL Reference Manual, Optimizing UPDATE Statements
- MySQL Reference Manual, Optimizing DELETE Statements
- MySQL Reference Manual, Optimizing InnoDB Transaction Management
- MySQL Reference Manual, InnoDB and Replication
