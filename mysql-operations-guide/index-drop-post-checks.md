# MySQL 인덱스 삭제 후 쿼리 플랜 검증 방법

## 이 문서는 무엇을 설명하나

이 문서는 인덱스를 삭제한 뒤 실제로 안전한지 확인하는 방법을 정리합니다. 핵심은 "삭제가 성공했는가"보다 "삭제 후 쿼리와 DML 경로가 나빠지지 않았는가"를 검증하는 것입니다.

## 먼저 한 줄로 요약하면

- 인덱스 삭제 후에는 `EXPLAIN`, `EXPLAIN ANALYZE`, rows examined, latency, lock footprint, slow query, replication lag를 같이 봐야 합니다.
- 삭제 전후를 비교하지 않으면 문제를 놓치기 쉽습니다.

## 1. 삭제 전 기준선 저장하기

삭제 전부터 아래를 저장해 두는 것이 중요합니다.

- 핵심 쿼리의 `EXPLAIN`
- 가능하면 `EXPLAIN ANALYZE`
- p95/p99 latency
- rows examined
- lock wait / deadlock
- replica lag

비교 기준이 없으면 삭제 후 문제가 생겨도 "원래 그랬는지" 판단하기 어렵습니다.

## 2. 가장 먼저 볼 것: EXPLAIN

삭제 후 핵심 쿼리의 실행 계획을 다시 봅니다.

```sql
EXPLAIN SELECT ...;
```

### 확인 포인트

- 어떤 `key`를 선택했는가
- `possible_keys`가 어떻게 바뀌었는가
- `type`이 나빠졌는가
- `rows` 추정치가 급증했는가
- `Using filesort`, `Using temporary`가 새로 생겼는가

## 3. 가능하면 EXPLAIN ANALYZE까지 보기

```sql
EXPLAIN ANALYZE SELECT ...;
```

이걸로 실제 실행 시간을 더 현실적으로 볼 수 있습니다.

### 확인 포인트

- actual time 증가 여부
- actual rows 증가 여부
- 예상보다 넓은 scan이 발생하는지

## 4. rows examined 변화 보기

인덱스 삭제 후 가장 흔한 문제 중 하나는 rows examined 급증입니다.

### 왜 중요한가

- SELECT가 느려질 수 있음
- `UPDATE`/`DELETE`가 더 많은 row를 건드리며 lock footprint가 커질 수 있음
- replica apply에도 부정적 영향을 줄 수 있음

### 위험 신호

- 삭제 전보다 rows examined가 크게 증가함
- 특정 API만 유독 더 많은 row를 읽기 시작함

## 5. latency 보기

운영에서는 결국 사용자 latency가 중요합니다.

### 확인할 지표

- API latency
- DB query latency
- commit latency
- timeout rate

### 특히 확인할 것

- 삭제 대상 인덱스를 타던 endpoint만 느려졌는가
- 전체 서비스보다 특정 기능만 악화됐는가

## 6. lock footprint 보기

인덱스를 삭제하면 SELECT뿐 아니라 DML 경로도 바뀔 수 있습니다.

예를 들어:

- `UPDATE`가 더 넓은 범위를 스캔함
- `DELETE`가 더 많은 row를 잠그게 됨
- lock wait와 deadlock이 늘어남

### 확인할 것

- lock wait 증가 여부
- deadlock 증가 여부
- 특정 배치 job의 실행 시간 증가 여부

## 7. slow query 신호 보기

삭제 후에는 slow query와 rows examined 비율을 꼭 봅니다.

### 위험 신호

- 새 slow query가 나타남
- 기존 query가 더 자주 slow query로 잡힘
- rows examined / rows sent 비율이 급증함

### 실무 포인트

- 평소 조용하던 query가 갑자기 slow log에 자주 보이면 의심해야 합니다.

## 8. replication lag 보기

삭제 후 primary만 보면 안 됩니다.

### 확인할 것

- replica lag 증가 여부
- slave/applier thread 지연 여부
- replica read latency 증가 여부

### 왜 생기나

- 삭제 후 변경된 쿼리 계획이 replica에도 반영됨
- replica가 더 약한 하드웨어면 문제가 더 빨리 드러날 수 있음

## 9. fallback 검증

문제가 생기면 바로 복구할 수 있어야 합니다.

### 준비할 것

- 원복용 `CREATE INDEX` 문
- 인덱스 컬럼 순서 기록
- online 생성 가능 여부 확인
- 원복 후 다시 확인할 쿼리 목록

예시:

```sql
CREATE INDEX idx_name ON your_table (col1, col2);
```

### fallback 판단 기준 예시

- 핵심 API latency가 허용 범위를 넘음
- rows examined 급증
- lock wait/deadlock 급증
- replica lag가 임계값 초과

## 10. 삭제 후 검증 체크리스트

- 삭제 전후 `EXPLAIN` 비교 완료
- 가능하면 `EXPLAIN ANALYZE` 비교 완료
- rows examined 변화 확인 완료
- 핵심 API latency 확인 완료
- lock wait/deadlock 변화 확인 완료
- slow query 증가 여부 확인 완료
- replica lag 확인 완료
- fallback SQL 준비 완료

## 11. 한 줄 결론

- 인덱스 삭제 후 검증의 핵심은 "query plan이 바뀌었는가"보다 "그 변화가 실제 운영 증상으로 이어졌는가"까지 보는 것입니다.

## 참고 자료

- MySQL Reference Manual, EXPLAIN Statement
- MySQL Reference Manual, Online DDL Operations
- MySQL Reference Manual, Slow Query Log
- MySQL Reference Manual, Replication Status
