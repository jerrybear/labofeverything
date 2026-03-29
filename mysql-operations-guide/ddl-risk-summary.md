# MySQL DDL 위험도 요약표

## 이 문서는 무엇을 설명하나

이 문서는 운영 중 자주 마주치는 MySQL DDL 작업을 한 장으로 빠르게 비교하기 위한 요약표입니다. 특히 live DML이 들어오는 환경에서 무엇이 상대적으로 가볍고, 무엇이 위험한지 빠르게 판단할 수 있도록 정리했습니다.

## 먼저 한 줄로 요약하면

- `INSTANT`에 가까운 변경은 상대적으로 안전합니다.
- 인덱스 추가는 가능하지만 busy table에서는 비싸질 수 있습니다.
- 인덱스 삭제는 작업보다 삭제 후 실행 계획 변화가 더 위험할 수 있습니다.
- `MODIFY COLUMN`, `PRIMARY KEY 변경`은 운영상 고위험군으로 보는 편이 안전합니다.
- `partition drop`은 대량 `DELETE`보다 유리하지만 metadata lock과 replica 영향은 여전히 봐야 합니다.

## 빠른 위험도 표

| 작업 | 일반 위험도 | DML 유입 시 주의 포인트 | 운영 체감 포인트 | 기본 운영 판단 |
| --- | --- | --- | --- | --- |
| 인덱스 추가 | 중간 | 동시 DML 반영 비용 증가 | write latency, I/O, replica lag | 가능하지만 traffic를 봐야 함 |
| 인덱스 삭제 | 중간 | 실행 계획 변화 | scan 증가, lock footprint 확대 | 삭제 후 쿼리 계획 검증 필수 |
| ADD COLUMN (`INSTANT`) | 낮음 | metadata lock 짧게 존재 | 거의 무증상 또는 아주 짧은 stall | 비교적 안전 |
| ADD COLUMN (`INPLACE` 이상) | 중간~높음 | 내부 작업과 자원 사용 증가 | write throughput 저하, lag 증가 | 알고리즘 먼저 확인 |
| MODIFY COLUMN | 높음 | 리빌드 가능성 큼 | ALTER 장기 실행, temp/I/O 사용 증가 | 고위험으로 취급 |
| PRIMARY KEY 추가/변경 | 매우 높음 | clustered index 구조 영향 | 디스크/I/O 증가, replica 지연 | 점검 창 강하게 필요 |
| partition drop | 낮음~중간 | metadata lock, replica apply 영향 | 짧은 stall, lag 증가 | 대량 delete보다 우수 |

## 작업별 핵심 포인트

### 1) 인덱스 추가

- DML이 계속 들어오면 새 인덱스 반영 비용이 같이 붙음
- hot table에서는 write latency와 flush pressure가 먼저 체감될 수 있음
- replica가 약하면 lag가 더 빨리 튈 수 있음

### 2) 인덱스 삭제

- 삭제 작업 그 자체보다 삭제 후 plan change가 더 위험할 수 있음
- 기존에 인덱스를 타던 update/delete가 넓은 scan으로 바뀌면 lock 범위가 커질 수 있음

### 3) ADD COLUMN

- `INSTANT`면 매우 유리함
- 하지만 실제 알고리즘이 `INSTANT`가 아니면 예상보다 비싸질 수 있음
- "컬럼 하나 추가"라는 표현만 보고 가볍다고 판단하면 위험함

### 4) MODIFY COLUMN

- 운영에서 가장 자주 과소평가되는 작업 중 하나
- 타입 변경, 길이 축소, NULL 속성 변경 등은 리빌드성 작업이 될 수 있음
- busy table에서는 장시간 부하와 lag를 만들 가능성이 큼

### 5) PRIMARY KEY 변경

- InnoDB 구조상 가장 민감한 변경 중 하나
- 단순한 인덱스 작업처럼 보면 안 됨
- 대체로 별도 점검 창, fallback 전략, replica 영향 검토가 필요함

### 6) partition drop

- row-by-row delete보다 훨씬 유리함
- undo, purge 부담을 크게 줄일 수 있음
- 그래도 metadata lock과 replica 지연 가능성은 봐야 함

## 운영자가 바로 쓰는 판단 기준

### 비교적 안전하게 시작할 수 있는 것

- `INSTANT` ADD COLUMN
- 비교적 단순한 인덱스 추가
- 잘 준비된 partition drop

### 특히 신중해야 하는 것

- MODIFY COLUMN
- PRIMARY KEY 추가 또는 변경
- 대량 batch DML과 겹치는 인덱스 작업

### 작업보다 결과가 더 위험한 것

- 인덱스 삭제

## 실행 전 체크리스트

- long transaction 없음
- hot table 여부 확인
- 작업 알고리즘 확인
- replica lag 여유 확인
- 핵심 쿼리 계획 확인
- rollback 또는 stop 전략 준비

## 한 줄 결론

- 운영에서는 "DDL이 가능한가"보다 "이 DDL을 live DML 위에서 버틸 수 있는가"를 먼저 봐야 합니다.

## 참고 자료

- MySQL Reference Manual, Online DDL Operations
- MySQL Reference Manual, Online DDL Performance and Concurrency
