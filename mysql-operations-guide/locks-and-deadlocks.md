# MySQL 락과 데드락 가이드

## 이 가이드는 누구를 위한 문서인가

이 문서는 대기 중인 트랜잭션, blocking chain, deadlock 때문에 MySQL이 갑자기 느려지는 상황을 이해해야 하는 운영자와 백엔드 엔지니어를 위한 가이드입니다.

## 핵심 요약

- InnoDB에서는 많은 운영 장애가 실제로는 CPU 문제가 아니라 lock 문제입니다.
- deadlock 자체는 버그가 아닙니다. 애플리케이션이 재시도를 전제로 설계해야 하는 정상적인 동시성 이벤트입니다.
- 운영상 더 위험한 것은 잦은 lock wait, 긴 트랜잭션, 넓은 범위 스캔 때문에 충돌 영역이 커지는 상황입니다.
- 운영자는 보통 blocker를 찾고, 트랜잭션을 짧게 만들고, 접근 순서와 인덱스를 개선하면서 문제를 해결합니다.

## 1. 운영자가 신경 써야 할 락 종류

### 공유 락과 배타 락

행 수준에서 InnoDB는 주로 다음 락을 사용합니다.

- 읽기 보호가 필요한 경우의 shared lock
- update와 delete를 위한 exclusive lock

실무에서는 `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE`가 운영상 lock contention의 출발점인 경우가 많습니다.

### Intention lock

이는 row locking을 보조하는 테이블 수준 신호입니다. 운영자가 직접 조정하는 대상은 아니지만, table lock과 row lock이 함께 얽힐 때 의미가 있습니다.

### Gap lock

gap lock은 인덱스 레코드 사이 범위를 보호하며, 해당 범위로의 insert를 막습니다.

운영상 gap lock이 중요한 이유:

- 처음엔 관련 없어 보이는 insert도 막을 수 있음
- 범위 조건이 있을 때 자주 나타남
- `REPEATABLE READ`에서 더 두드러짐

### Next-key lock

next-key lock은 row lock과 그 앞 구간의 gap lock이 결합된 형태입니다.

운영상 중요한 이유는 phantom row를 막는 데 도움이 되지만, 범위 쿼리에서 충돌 영역을 더 넓히기 때문입니다.

### Metadata lock

metadata lock은 테이블 정의를 보호합니다.

운영상 중요한 이유:

- 긴 트랜잭션이 metadata lock을 생각보다 오래 잡고 있을 수 있음
- `ALTER TABLE` 같은 DDL이 일반 트랜잭션 뒤에서 대기할 수 있음
- 눈에 보이는 장애의 원인이 사실은 잊고 있던 트랜잭션 때문에 대기 중인 DDL일 수 있음

## 2. lock contention은 보통 왜 생기나

운영 환경에서 흔한 원인:

- 두 트랜잭션이 같은 hot row를 동시에 갱신함
- 넓은 조건으로 range update 또는 delete를 수행함
- 인덱스 부족 때문에 많은 행을 스캔하며 락 범위도 커짐
- 트랜잭션마다 테이블 접근 순서가 다름
- 애플리케이션 코드가 네트워크 호출이나 비즈니스 로직 동안 트랜잭션을 열어둠
- 배치 작업이 사용자 트래픽과 경쟁함

실무에서는 단순히 동시성이 높아서가 아니라, 너무 많은 행을 너무 오래 잡고 있어서 문제가 되는 경우가 많습니다.

## 3. deadlock이 실제로 의미하는 것

deadlock은 트랜잭션 A가 트랜잭션 B를 기다리고, 동시에 트랜잭션 B가 트랜잭션 A 또는 더 긴 사이클 안의 다른 트랜잭션을 기다릴 때 발생합니다.

InnoDB는 이 순환을 감지하고 하나의 트랜잭션을 victim으로 골라 rollback합니다.

운영상 의미:

- 가끔 한 번 발생하는 deadlock은 정상 범주일 수 있음
- deadlock이 자주 일어나면 설계 또는 workload shape 문제를 의심해야 함
- deadlock 하나의 존재보다 deadlock 빈도가 더 중요함

## 4. 흔한 deadlock 패턴

### 반대 순서로 같은 행 접근

트랜잭션 1은 row 10을 갱신한 뒤 row 20을 갱신합니다.
트랜잭션 2는 row 20을 갱신한 뒤 row 10을 갱신합니다.

가장 흔한 deadlock 패턴 중 하나입니다.

### 같은 테이블, 다른 접근 순서

한 코드 경로는 `accounts`를 먼저 갱신하고 `orders`를 나중에 갱신하는데, 다른 경로는 `orders`를 먼저 갱신한 뒤 `accounts`를 만집니다.

### 범위 스캔과 insert 충돌

범위를 스캔하며 락을 잡는 트랜잭션과, 그 범위 안으로 insert하려는 다른 트랜잭션이 충돌할 수 있습니다.

### 대형 배치 작업과 사용자 트래픽의 충돌

유지보수용 `UPDATE` 또는 `DELETE`가 많은 짧은 사용자 트랜잭션과 겹치면서 deadlock을 반복적으로 만들 수 있습니다.

## 5. 운영자가 보게 되는 증상

- 일부 엔드포인트의 application latency만 상승함
- lock wait timeout 오류가 나타남
- 애플리케이션에서 `ERROR 1213` 같은 deadlock 오류가 발생함
- `SHOW PROCESSLIST`에 대기 상태 세션이 많이 보임
- DB 호스트는 대체로 멀쩡해 보이는데 사용자들은 느리다고 느낌
- 긴 트랜잭션과 무거운 apply 작업 때문에 replica lag도 함께 나타날 수 있음

## 6. 먼저 무엇을 봐야 하나

### 현재 대기 중인 세션

```sql
SHOW FULL PROCESSLIST;
```

다음을 봅니다.

- 오래 열린 트랜잭션
- lock을 기다리는 세션
- 같은 테이블이나 키 범위를 반복적으로 건드리는 statement

### InnoDB 상태

```sql
SHOW ENGINE INNODB STATUS\G
```

이 명령은 여전히 다음 정보를 가장 빨리 보여주는 도구 중 하나입니다.

- 현재 lock wait
- 최근 감지된 deadlock
- 어떤 트랜잭션이 얼마나 오래 열려 있는지

### blocker와 waiter 관계

MySQL 버전과 instrumentation에 따라 `performance_schema` 또는 `information_schema` view를 사용해 다음을 볼 수 있습니다.

- 대기 중인 transaction ID
- 막고 있는 transaction ID
- lock mode
- 관련 테이블과 인덱스

예시:

```sql
SELECT * FROM performance_schema.data_lock_waits;
SELECT * FROM performance_schema.data_locks;
```

## 7. 운영자가 안전하게 대응하는 방법

### 하나의 쿼리가 많은 쿼리를 막고 있는 경우

- 먼저 blocker를 식별함
- 그것이 사용자 트랜잭션인지, 배치 작업인지, 마이그레이션인지 확인함
- 무작정 kill하지 말고, 가장 작거나 가장 안전하게 중단할 수 있는 작업을 우선 고려함

정말 중단해야 한다면:

```sql
KILL QUERY <process_id>;
```

전체 연결을 끊는 편이 더 안전한 경우에만 `KILL <process_id>`를 사용합니다.

### deadlock이 가끔 발생하는 경우

- 정상적인 동시성 동작으로 받아들임
- 애플리케이션이 트랜잭션 재시도를 하는지 확인함
- deadlock 빈도가 허용 범위인지 검토함

### deadlock이 자주 발생하는 경우

- 필요하면 deadlock 로그를 더 넓게 남김
- 반복되는 테이블, 인덱스, 쿼리 패턴을 조사함
- 코드 경로별 transaction order를 비교함
- 유지보수 작업의 batch size를 줄임

유용한 설정:

```sql
SET GLOBAL innodb_print_all_deadlocks = ON;
```

## 8. 대체로 잘 통하는 예방 패턴

### 트랜잭션을 짧게 유지하기

가장 중요한 운영 원칙은 트랜잭션을 작고 빠르게 유지하는 것입니다.

- 한 트랜잭션 안에서 하는 일을 줄임
- 트랜잭션이 열린 상태에서 외부 시스템을 기다리지 않음
- 비즈니스 규칙이 허용하면 더 빨리 commit함

### 항상 같은 순서로 접근하기

여러 코드 경로가 같은 논리 엔터티를 건드린다면, 가능하면 같은 순서로 접근해야 합니다.

### 더 좋은 인덱스 사용하기

좋은 인덱스는 스캔 행 수를 줄여 lock footprint를 줄입니다.

### batch size 줄이기

큰 `UPDATE`와 `DELETE`는 lock duration과 lock area를 모두 키웁니다. 더 작은 chunk가 대체로 안전합니다.

### 필요할 때 isolation level 재검토하기

일부 시스템에서는 `READ COMMITTED`가 `REPEATABLE READ`보다 gap-lock 관련 contention을 줄일 수 있습니다. 다만 동시성 의미가 바뀌므로 신중한 평가가 필요합니다.

## 9. 추천 대응 플레이북

### lock wait가 급증할 때

1. hot table과 blocker를 찾음
2. 최근에 새 배포나 배치 작업이 시작됐는지 확인함
3. 인덱스 부족 때문에 scan 범위가 넓어진 것은 아닌지 점검함
4. 경쟁 중인 batch work를 줄이거나 멈춤
5. 비즈니스 영향이 정당화될 때만 blocker를 kill함

### deadlock이 급증할 때

1. 최신 deadlock 상세를 확인함
2. statement 형태와 테이블별로 deadlock을 묶어봄
3. 애플리케이션 코드 경로 간 접근 순서를 비교함
4. 트랜잭션 크기를 줄이고 안전하게 재시도함
5. 반복 패턴을 엔지니어링 후속 조치용으로 기록함

## 10. 애플리케이션이 갖춰야 할 기대 사항

애플리케이션은 다음을 전제로 설계되어야 합니다.

- deadlock victim이 되면 안전하게 재시도할 수 있어야 함
- 가능하면 트랜잭션을 idempotent하게 유지해야 함
- 느린 외부 호출 동안 트랜잭션을 열어두지 않아야 함
- lock timeout과 deadlock 메트릭을 노출해야 함

애플리케이션이 deadlock을 예상하면 deadlock은 견딜 수 있는 이벤트입니다. 재시도 동작이 없거나 트랜잭션 설계가 나쁘면 incident가 됩니다.

## 11. 빠른 체크리스트

대응 전:

- blocker 식별
- victim 경로나 hot query 식별
- 비즈니스 영향 확인

대응 중:

- lock wait 관찰
- application latency 관찰
- deadlock rate 관찰

대응 후:

- waiting chain이 해소됐는지 확인
- transaction 수가 정상화됐는지 확인
- root cause와 예방 변경 사항 기록

## 참고 자료

- MySQL Reference Manual, InnoDB Locking
- MySQL Reference Manual, Deadlocks in InnoDB
- MySQL Reference Manual, Deadlock Detection
- MySQL Reference Manual, How to Minimize and Handle Deadlocks
