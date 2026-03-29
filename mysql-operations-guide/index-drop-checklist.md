# MySQL 인덱스 삭제 전 체크리스트

## 이 문서는 무엇을 설명하나

이 문서는 운영 환경에서 인덱스를 삭제하기 전에 반드시 확인해야 할 항목을 체크리스트 형태로 정리합니다. 목적은 "공간을 줄이기 위해 인덱스를 지웠다가 성능 문제를 만드는 일"을 줄이는 것입니다.

## 먼저 한 줄로 요약하면

- 인덱스 삭제는 보통 인덱스 추가보다 가볍지만, 삭제 후 쿼리 계획 변화가 더 위험할 수 있습니다.
- 삭제 전에 반드시 `누가 쓰는지`, `대체 인덱스가 있는지`, `replica가 버틸지`, `롤백이 가능한지`를 확인해야 합니다.

## 1. 사용 여부 확인

### 확인 질문

- 이 인덱스를 실제로 타는 쿼리가 있는가
- 항상 쓰는 인덱스는 아니더라도, 배치/리포트/월말 작업에서 쓰는가
- application code나 ORM이 암묵적으로 기대하는 인덱스인가

### 실무 체크 포인트

- slow query log
- query digest / application query logs
- `EXPLAIN` 또는 `EXPLAIN ANALYZE`
- `sys.schema_unused_indexes`
- `performance_schema.table_io_waits_summary_by_index_usage`

### 주의

- `unused`로 보인다고 바로 지우면 안 됩니다.
- 관찰 기간이 짧으면 월간 배치나 특정 시간대 작업을 놓칠 수 있습니다.

## 2. 중복/겹침 인덱스 여부 확인

### 꼭 확인할 것

- 완전 중복 인덱스인지
- 더 긴 복합 인덱스가 같은 역할을 이미 하는지
- left-prefix 원칙상 단일 인덱스가 불필요한지

예를 들어:

```sql
INDEX idx_user_id (user_id)
INDEX idx_user_id_created_at (user_id, created_at)
```

이 경우 `idx_user_id`는 일부 상황에서 중복 후보가 될 수 있습니다.

### 하지만 바로 삭제하면 안 되는 경우

- 짧은 인덱스가 더 작아서 optimizer가 더 자주 고를 수 있음
- 정렬, covering, selectivity 측면에서 역할이 다를 수 있음

## 3. 제약조건 확인

### 절대 먼저 봐야 하는 것

- UNIQUE 제약을 보장하는 인덱스인가
- PRIMARY KEY와 관련된 인덱스인가
- foreign key 검증에 필요한 인덱스인가

### 위험 포인트

- UNIQUE 인덱스를 지우면 중복 데이터가 들어올 수 있음
- foreign key에 필요한 인덱스를 지우려 하면 MySQL이 막거나, 다른 경로에 영향이 생길 수 있음

## 4. 쿼리 의존성 확인

### 꼭 확인할 쿼리 종류

- 사용자 요청을 직접 받는 API 쿼리
- 백오피스 조회
- 배치 `UPDATE`/`DELETE`
- 정렬이 붙은 조회
- 범위 조회
- covering index에 의존하는 조회

### 검증 질문

- 이 인덱스를 지우면 어떤 인덱스를 대신 타게 되는가
- 아니면 full scan으로 떨어지는가
- `UPDATE`/`DELETE`의 rows examined가 급증할 가능성이 있는가

## 5. DML 영향 확인

인덱스를 지우면 쓰기 오버헤드는 줄 수 있습니다. 하지만 읽기/조건 탐색 경로가 나빠지면 오히려 운영은 더 아파질 수 있습니다.

### 삭제 전에 봐야 할 것

- 이 테이블은 read-heavy인가, write-heavy인가
- 인덱스 삭제로 `INSERT`/`UPDATE`/`DELETE` 비용 절감이 의미 있는가
- 반대로 중요한 조회와 변경 쿼리가 더 넓은 scan으로 바뀌지 않는가

### 핵심 포인트

- 쓰기 부담을 줄이기 위해 인덱스를 지우더라도
- 그 결과로 `UPDATE`/`DELETE`가 더 많은 row를 스캔하면 lock footprint가 커질 수 있습니다.

## 6. metadata lock 확인

인덱스 삭제 자체는 보통 가벼운 편이지만 DDL이기 때문에 metadata lock 구간은 존재합니다.

### 삭제 전 꼭 확인

- long transaction 존재 여부
- idle in transaction 세션 존재 여부
- 같은 테이블에 대해 기다리는 DDL/쿼리가 있는지

### 기본 확인 SQL

```sql
SHOW FULL PROCESSLIST;
SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX\G
```

## 7. replication 영향 확인

### 꼭 확인할 것

- replica lag 여유가 있는가
- replica가 primary보다 약한 하드웨어인가
- replica에 read workload가 많이 붙어 있는가

### 운영 감각

- 인덱스 삭제 작업 자체보다도
- 삭제 후 쿼리 계획 변화가 replica apply나 replica read 성능을 더 나쁘게 만들 수 있습니다.

## 8. 롤백 계획 준비

삭제하기 전에 반드시 원복용 인덱스 생성문을 준비합니다.

```sql
CREATE INDEX idx_name ON your_table (col1, col2);
```

### 체크 포인트

- 인덱스 이름, 컬럼 순서, 고유성 포함 여부를 기록했는가
- 원복 시 online 옵션을 어떻게 줄지 정했는가
- 원복 예상 시간이 어느 정도인지 감이 있는가

## 9. 실행 전 최종 체크리스트

- 인덱스 사용 여부를 충분한 기간으로 확인함
- 중복/겹침 인덱스인지 확인함
- UNIQUE/FK 의존성을 확인함
- 핵심 쿼리 `EXPLAIN`을 확보함
- DML 영향과 lock footprint 변화를 검토함
- long transaction이 없는지 확인함
- replica lag 여유를 확인함
- rollback용 `CREATE INDEX` 문을 준비함

## 10. 한 줄 결론

- 인덱스 삭제는 "공간 정리" 작업이 아니라 "실행 계획 변경" 작업으로 생각하는 편이 안전합니다.

## 참고 자료

- MySQL Reference Manual, DROP INDEX Statement
- MySQL Reference Manual, Online DDL Operations
- MySQL Reference Manual, sys schema redundant/unused indexes
