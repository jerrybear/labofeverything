# Grafana의 MySQL 트랜잭션 히스토리 지표 이해하기

## 이 문서는 무엇을 설명하나

이 문서는 Grafana에서 MySQL 지표로 보이는 `트랜잭션 히스토리`가 보통 무엇을 뜻하는지, 왜 올라가는지, 어느 수준부터 주의해야 하는지, 그리고 같이 봐야 할 연관 지표는 무엇인지 운영 관점에서 설명합니다.

## 먼저 한 줄로 설명하면

Grafana에서 보이는 MySQL `트랜잭션 히스토리`는 대개 InnoDB의 `History list length`를 뜻합니다. 내부적으로는 `trx_rseg_history_len` 계열 값으로 노출되는 경우가 많습니다.

쉽게 말하면:

- 아직 purge되지 못한 변경 이력의 길이
- undo log 안에 남아 있는 과거 row 버전의 양
- purge thread가 나중에 정리해야 할 작업량

이 값은 MVCC와 직접 연결됩니다.

## 1. 이 값이 정확히 무엇인가

InnoDB는 `UPDATE`나 `DELETE`가 발생하면 기존 row 버전을 undo log에 보관합니다. 이유는 아직 열려 있는 다른 트랜잭션이 과거 시점의 일관된 데이터를 읽어야 할 수 있기 때문입니다.

그래서 `History list length`는 다음 의미로 이해하면 됩니다.

- 현재 즉시 지울 수 없는 과거 변경 이력이 얼마나 쌓였는가
- purge가 아직 처리하지 못한 undo 기반 작업이 얼마나 남았는가
- 긴 트랜잭션이나 무거운 DML 때문에 MVCC 부담이 얼마나 누적됐는가

운영에서는 이 값을 "InnoDB가 아직 뒤처지지 않고 잘 청소하고 있는가"를 보는 지표로 생각하면 이해가 쉽습니다.

## 2. 왜 `UPDATE`와 `DELETE`에서 특히 잘 올라가나

사용자가 궁금해한 첫 번째 포인트입니다.

### `UPDATE`

`UPDATE`가 발생하면 InnoDB는 새 값만 쓰는 것이 아니라 이전 값을 undo log에 남깁니다.

그 이유는:

- rollback이 가능해야 하고
- 아직 끝나지 않은 다른 트랜잭션은 이전 버전을 읽을 수 있어야 하기 때문입니다

즉, update가 많아질수록 과거 버전이 더 많이 생기고, purge가 이를 제때 정리하지 못하면 history list가 올라갑니다.

### `DELETE`

`DELETE` 역시 단순히 즉시 완전 삭제되는 것이 아닙니다.

- 삭제 정보가 undo에 남고
- 다른 트랜잭션이 그 row의 이전 상태를 볼 수 있어야 하면 바로 정리되지 못하고
- purge가 나중에 후속 정리를 해야 합니다

그래서 대량 delete는 history list를 빠르게 키우는 대표 작업입니다.

### 특히 더 잘 올라가는 상황

다음 상황에서는 `UPDATE`/`DELETE`가 history list를 더 빠르게 키웁니다.

- 긴 트랜잭션이 열려 있음
- `REPEATABLE READ`에서 오래 유지되는 읽기 트랜잭션이 있음
- 대량 batch update/delete가 들어옴
- purge thread가 쓰기량을 따라가지 못함
- `mysqldump --single-transaction` 같은 장시간 읽기 작업이 존재함

한 줄로 요약하면:

- `UPDATE`와 `DELETE`는 과거 버전을 만들고
- 긴 트랜잭션은 그 과거 버전을 오래 잡아두고
- purge가 못 따라가면 transaction history 값이 계속 올라갑니다

## 3. 어느 정도부터 주의 신호로 봐야 하나

사용자가 궁금해한 두 번째 포인트입니다.

이 값은 절대적인 정답 숫자 하나가 있는 지표는 아닙니다. 시스템의 쓰기량, purge 속도, 트랜잭션 패턴에 따라 다르게 해석해야 합니다. 다만 운영상 감각적으로는 아래처럼 볼 수 있습니다.

| 수준 | 해석 |
| --- | --- |
| 수백 ~ 수천 | 대체로 건강한 편 |
| 수천 ~ 수만 | 증가 추세를 보기 시작해야 함 |
| 수만 ~ 수십만 | 긴 트랜잭션 또는 purge 지연 의심 |
| 수십만 이상이 오래 유지 | 분명한 운영 경고 신호 |

중요한 것은 숫자 자체보다 **추세**입니다.

### 괜찮을 수 있는 경우

- 짧은 배치 작업 직후 잠깐 상승함
- 잠시 올랐다가 다시 내려옴
- 사용자 지연 시간이나 replica lag에 영향이 없음

### 위험하게 봐야 하는 경우

- 계속 우상향함
- 높은 값이 오래 유지됨
- 동시에 lock wait, write latency, replica lag도 같이 나빠짐
- 오래 열린 transaction이 함께 관찰됨

즉, 이 값은 "얼마나 높나"보다 "얼마나 오래 높게 유지되나"가 더 중요합니다.

## 4. 운영에서는 어떤 영향이 생기나

이 값이 높고 오래 유지되면 대체로 다음 문제가 동반됩니다.

- undo 관련 부담 증가
- purge backlog 증가
- 일부 query가 더 많은 row version을 따라가느라 느려질 수 있음
- 큰 write 부하 후 recovery나 shutdown이 더 무거워질 수 있음
- 대량 DML, 긴 transaction, replica lag 문제와 함께 보이는 경우가 많음

그래서 이 지표는 단순한 내부 값이 아니라, "지금 MySQL이 과거 이력을 치우지 못하고 쌓아두고 있는가"를 보여주는 운영 지표라고 보면 됩니다.

## 5. Grafana에서 같이 봐야 할 연관 지표 5개

사용자가 궁금해한 세 번째 포인트입니다.

이 지표 하나만 보면 원인을 놓치기 쉽습니다. 운영에서는 아래 5개를 같이 보는 것이 좋습니다.

### 1) active transaction 수와 오래 열린 transaction

가장 먼저 봐야 할 지표입니다.

- history list가 오르는데 오래 열린 transaction이 있으면 원인일 가능성이 큼
- 값이 계속 높다면 purge가 기다리고 있을 가능성이 큼

### 2) lock wait / deadlock

history list 상승이 대량 update/delete나 긴 transaction과 함께 발생하면 lock 문제도 같이 생기는 경우가 많습니다.

- lock wait 증가
- deadlock 증가
- blocking chain 증가

### 3) write throughput 또는 rows changed

트랜잭션 히스토리 값이 오른다고 해서 항상 긴 transaction 때문만은 아닙니다.

- 초당 update/delete 양이 급증했는지
- bulk job이 시작됐는지
- 쓰기량이 purge 속도를 넘어섰는지

를 함께 봐야 합니다.

### 4) replica lag

history list 상승은 대량 DML과 동반되는 경우가 많고, 그런 작업은 replica에도 그대로 부담을 줍니다.

- source에서 대량 변경이 있었는지
- replica가 apply를 따라가고 있는지
- stale read 위험이 커지고 있지는 않은지

를 같이 봐야 합니다.

### 5) query latency 또는 commit latency

운영자 입장에서는 결국 사용자 영향이 중요합니다.

- history list는 높지만 실제 latency 영향이 없는지
- 아니면 commit latency, API latency, timeout이 같이 올라가는지
- 시스템이 실제로 아픈 상태인지

를 같이 봐야 합니다.

## 6. 직접 확인할 때 유용한 SQL

### InnoDB 상태에서 확인

```sql
SHOW ENGINE INNODB STATUS\G
```

여기서 `History list length`를 확인할 수 있습니다.

### 오래 열린 트랜잭션 찾기

```sql
SELECT *
FROM INFORMATION_SCHEMA.INNODB_TRX\G
```

특히 다음을 봅니다.

- 언제 시작됐는지
- 얼마나 오래 열려 있는지
- 현재 어떤 상태인지

### 더 직접적으로 metric 보기

환경에 따라 아래 metric을 직접 볼 수도 있습니다.

```sql
SELECT COUNT
FROM INFORMATION_SCHEMA.INNODB_METRICS
WHERE NAME = 'trx_rseg_history_len';
```

## 7. 운영자가 실무에서 해석하는 법

실무에서는 아래처럼 해석하면 됩니다.

- 값이 잠깐 올랐다가 잘 내려오면 대체로 괜찮음
- 값이 계속 오른다면 purge가 밀리거나 긴 transaction이 있는 것
- 값이 높은데 latency, lock wait, lag도 같이 나빠지면 적극 대응 필요
- 항상 숫자 하나보다 추세와 동반 증상을 함께 봐야 함

운영상 가장 흔한 실제 원인은:

1. 잊고 열어둔 긴 transaction
2. 대량 `UPDATE`/`DELETE` batch
3. purge가 못 따라가는 write burst

## 8. 한 번에 기억할 요약

- Grafana의 MySQL `트랜잭션 히스토리`는 대개 InnoDB `History list length`
- 이는 아직 purge되지 못한 과거 row version의 양을 뜻함
- `UPDATE`/`DELETE`가 많고 긴 transaction이 있으면 잘 올라감
- 절대값보다 증가 추세와 유지 시간, 동반 증상이 중요함
- active transaction, lock wait, write throughput, replica lag, latency를 함께 봐야 함

## 참고 자료

- MySQL Reference Manual, history list
- MySQL Reference Manual, purge configuration
- MySQL Reference Manual, `INFORMATION_SCHEMA.INNODB_TRX`
