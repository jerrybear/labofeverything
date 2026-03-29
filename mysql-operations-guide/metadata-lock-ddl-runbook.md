# DDL 중 Metadata Lock 대기 대응 런북

## 이 문서는 무엇을 설명하나

이 문서는 MySQL에서 DDL 작업 중 `metadata lock` 대기가 발생해 서비스가 느려지거나 멈춘 것처럼 보일 때, 운영자가 어떤 순서로 확인하고 대응해야 하는지 정리한 런북입니다.

## 먼저 한 줄로 요약하면

- DDL 자체가 느린 경우보다, DDL이 metadata lock을 기다리면서 뒤의 일반 쿼리까지 연쇄적으로 막는 경우가 더 위험합니다.
- 가장 먼저 해야 할 일은 "누가 metadata lock을 오래 잡고 있는가"를 찾는 것입니다.
- 무작정 kill하기보다 blocker의 성격과 비즈니스 영향을 먼저 판단해야 합니다.

## 1. 이런 증상이 보이면 의심한다

- `ALTER TABLE`이 시작됐는데 끝나지 않음
- 일반 SELECT/UPDATE도 갑자기 느려짐
- `SHOW PROCESSLIST`에 `Waiting for table metadata lock`이 보임
- 배포 직후 특정 테이블 관련 요청만 멈춘 듯 보임
- DB CPU는 높지 않은데 애플리케이션 timeout이 증가함

## 2. 가장 먼저 할 일

### 1) 현재 상태 확인

```sql
SHOW FULL PROCESSLIST;
```

여기서 먼저 확인합니다.

- metadata lock을 기다리는 DDL이 있는지
- 오래 열린 transaction이 있는지
- 같은 테이블을 만지는 쿼리가 줄줄이 대기 중인지

### 2) blocker 후보 찾기

특히 아래 패턴을 먼저 의심합니다.

- 오래 열린 transaction
- idle in transaction 상태 세션
- 배치 작업 세션
- 트랜잭션 안에서 SELECT 후 오래 방치된 세션

## 3. 상세 확인 순서

### A. processlist 확인

```sql
SHOW FULL PROCESSLIST;
```

봐야 할 것:

- `Waiting for table metadata lock`
- 가장 오래 열린 세션
- 어떤 테이블을 잡고 있는지

### B. active transaction 확인

```sql
SELECT *
FROM INFORMATION_SCHEMA.INNODB_TRX\G
```

봐야 할 것:

- 언제 시작됐는지
- 얼마나 오래 열려 있는지
- 실제 작업 중인지, 사실상 방치된 상태인지

### C. metadata lock 정보 확인

가능하면 아래도 함께 봅니다.

```sql
SELECT *
FROM performance_schema.metadata_locks;
```

이걸로 waiting lock과 granted lock 관계를 더 구체적으로 볼 수 있습니다.

## 4. 운영자가 판단해야 하는 핵심 질문

### 질문 1) blocker는 누구인가

- 사용자 요청인가
- 배치 작업인가
- 마이그레이션/배포 세션인가
- 방치된 세션인가

### 질문 2) 지금 더 위험한 것은 무엇인가

- blocker를 죽이는 것
- DDL을 계속 기다리게 두는 것

### 질문 3) 이미 연쇄 대기가 생겼는가

- DDL 하나만 멈춘 상태인지
- 뒤의 일반 트래픽도 같이 막히고 있는지

이 세 질문을 먼저 정리하면 대응이 훨씬 안전해집니다.

## 5. 기본 대응 순서

### 1단계: blocker를 식별한다

- 가장 오래 열린 transaction부터 확인
- 대상 테이블을 건드린 세션을 찾음
- 같은 테이블에서 누가 granted lock을 갖고 있는지 찾음

### 2단계: DDL과 일반 트래픽 영향도를 비교한다

- 이미 사용자 요청까지 막히고 있으면 우선순위가 높음
- DDL만 기다리는 상태면 좀 더 신중하게 볼 수 있음

### 3단계: 가장 안전한 중단 대상을 고른다

- 방치된 세션이면 우선 정리 대상
- 재시도 가능한 배치면 중단 우선 검토
- 핵심 사용자 트랜잭션이면 더 신중하게 판단

### 4단계: 필요 시 중단한다

정말 필요할 때만:

```sql
KILL QUERY <process_id>;
```

전체 연결을 끊어야 할 때만:

```sql
KILL <process_id>;
```

## 6. 무작정 하면 안 되는 것

- blocker가 뭔지 모른 채 DDL 세션부터 kill하기
- 핵심 사용자 세션을 영향도 판단 없이 kill하기
- replica 상태를 안 보고 연속 DDL 재시도하기
- 배포 중 같은 테이블에 여러 DDL을 연달아 넣기

## 7. 자주 나오는 원인 패턴

### 1) idle in transaction

가장 흔합니다.

- 애플리케이션이 transaction을 열어둔 채 반환되지 않음
- 개발/운영 콘솔에서 수동 조회 후 commit을 안 함

### 2) 긴 배치 트랜잭션

- 대량 update/delete가 오래 열려 있음
- DDL이 시작하지 못하고 기다림

### 3) 배포와 배치가 겹침

- schema change와 maintenance batch가 같은 시간에 시작됨

### 4) 같은 테이블에 연속 DDL

- 하나의 DDL이 lock을 기다리는 동안 뒤의 작업도 줄줄이 쌓임

## 8. 복구 후 반드시 할 일

- waiting chain이 풀렸는지 확인
- 애플리케이션 latency가 정상화됐는지 확인
- replica lag가 튀지 않았는지 확인
- blocker 정체와 원인을 기록
- 같은 패턴 재발 방지책을 남김

## 9. 재발 방지 팁

- transaction을 짧게 유지
- idle in transaction을 감지하는 모니터링 추가
- DDL은 배치 시간과 겹치지 않게 스케줄링
- 동일 테이블 DDL은 순차적으로 수행
- 배포 전에 long transaction과 replica 상태를 확인

## 10. 초간단 대응 흐름

1. `SHOW FULL PROCESSLIST` 확인
2. `INFORMATION_SCHEMA.INNODB_TRX` 확인
3. blocker 식별
4. 비즈니스 영향 비교
5. 가장 안전한 세션만 정리
6. waiting chain 해소 여부 확인
7. 원인 기록

## 한 줄 결론

- metadata lock incident의 핵심은 "DDL이 느린가"가 아니라 "누가 시작을 못 하게 막고 있는가"를 찾는 것입니다.

## 참고 자료

- MySQL Reference Manual, Metadata Locking
- MySQL Reference Manual, Online DDL Performance and Concurrency
- MySQL Reference Manual, `INFORMATION_SCHEMA.INNODB_TRX`
