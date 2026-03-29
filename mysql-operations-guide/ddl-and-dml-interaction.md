# MySQL DDL 작업 중 DML 유입 영향 정리

## 이 문서는 무엇을 설명하나

이 문서는 운영 중 DML이 계속 들어오는 상황에서 다음 작업이 어떤 영향을 받는지 정리합니다.

- 인덱스 추가
- 인덱스 삭제
- partition drop
- ADD COLUMN
- MODIFY COLUMN
- PRIMARY KEY 변경

핵심은 "온라인 DDL이라도 완전히 공짜는 아니며, DML 유입이 계속되면 락, 지연, replication, 자원 사용량 측면에서 운영 영향이 커질 수 있다"는 점입니다.

## 먼저 한 줄로 요약하면

- DML이 들어오는 동안의 DDL은 보통 완전 차단보다는 "짧은 메타데이터 락 + 긴 백그라운드 작업 + 운영 부하 증가" 형태로 나타납니다.
- 사용자는 DDL 자체보다도 lock wait, write latency 증가, replica lag, flush pressure로 먼저 체감하는 경우가 많습니다.
- 특히 쓰기 유입이 많은 상태에서는 인덱스 추가, 인덱스 삭제, partition drop 모두 생각보다 비싸질 수 있습니다.

## 1. 왜 DML 유입이 DDL에 영향을 주는가

운영 중 DDL은 보통 다음 요소와 함께 움직입니다.

- metadata lock
- online DDL 중 발생하는 내부 작업
- DML이 계속 들어오면서 생기는 추가 변경 추적 비용
- binary log와 replication apply 비용

즉, `ALTER TABLE`이 시작되면 단순히 스키마만 바뀌는 것이 아니라, 이미 들어오고 있는 `INSERT`, `UPDATE`, `DELETE`와 충돌 없이 같이 굴러가도록 추가 비용을 지불하게 됩니다.

## 2. 공통적으로 먼저 알아야 할 것

### metadata lock은 짧아도 위험할 수 있음

온라인 DDL이라도 시작 시점과 끝 시점에는 metadata lock이 필요합니다.

운영상 중요한 점:

- 오래 열린 트랜잭션이 있으면 DDL 시작 자체가 대기할 수 있음
- DDL이 metadata lock을 기다리기 시작하면 뒤따르는 쿼리도 연쇄적으로 막힐 수 있음
- 사용자는 "ALTER TABLE은 안 끝나는데 일반 조회도 갑자기 느려졌다"처럼 느낄 수 있음

### online이라고 항상 무중단은 아님

`LOCK=NONE`, `ALGORITHM=INPLACE`, `ALGORITHM=INSTANT` 같은 옵션이 있어도 다음은 여전히 주의해야 합니다.

- 내부 리빌드 비용
- 정렬/로그/임시 공간 사용
- commit 직전 metadata 반영 구간
- replica에서의 동일 작업 재실행 또는 apply 지연

## 3. 인덱스 추가 시 DML 유입 영향

인덱스 추가는 운영에서 자주 하는 작업이지만, DML이 계속 들어오면 생각보다 부담이 커집니다.

### 운영에서 실제로 벌어지는 일

- 기존 데이터를 읽어 새 인덱스 구조를 만들어야 함
- 그 동안 들어오는 새로운 DML도 최종 인덱스 상태에 반영돼야 함
- 쓰기가 많을수록 내부 변경 추적 비용과 자원 사용량이 커짐

### 사용자가 체감하는 증상

- write latency 증가
- CPU 또는 I/O 사용량 증가
- flush pressure 증가
- replica lag 증가
- DDL 완료 직전 또는 시작 직전 짧은 멈춤

### 특히 위험한 상황

- hot table에 인덱스를 추가할 때
- 대량 `UPDATE`/`DELETE` batch가 동시에 돌 때
- 이미 lock wait가 많은 테이블일 때
- replica가 primary보다 약한 하드웨어일 때

### 운영 해석

인덱스 추가는 "DML이 가능하냐"보다 "DML이 계속 들어올 때도 시스템이 그 비용을 버티느냐"의 문제로 보는 것이 정확합니다.

## 4. 인덱스 삭제 시 DML 유입 영향

인덱스 삭제는 추가보다 가벼워 보이지만, 운영에서는 여전히 주의해야 합니다.

### 왜 영향이 생기나

- metadata 변경이 필요함
- optimizer가 즉시 다른 실행 계획으로 바뀔 수 있음
- 동시에 들어오는 DML과 SELECT가 다른 인덱스나 더 나쁜 경로를 타게 될 수 있음

### 운영에서 자주 보는 문제

- 삭제 직후 특정 쿼리 latency 급등
- 예상치 못한 table scan 증가
- DML이 기존보다 더 많은 row를 스캔하게 되면서 lock footprint 확대
- replica에서도 동일한 계획 변화가 발생

### 핵심 포인트

인덱스 삭제의 위험은 "삭제 작업 자체의 무거움"보다도, 삭제 후 쿼리 계획과 DML 경로가 나빠지는 데 있습니다.

즉, 운영에서는 인덱스를 지우기 전에 다음을 거의 반드시 봐야 합니다.

- 누가 그 인덱스를 읽고 있는지
- 삭제 후 어떤 인덱스를 대신 타게 될지
- update/delete 조건이 더 넓은 scan으로 바뀌지 않는지

## 5. Partition drop 시 DML 유입 영향

partition drop은 종종 "오래된 데이터 삭제를 빠르게 끝내는 방법"으로 쓰입니다. 실제로 row-by-row delete보다 훨씬 효율적일 수 있습니다.

하지만 운영에서는 여전히 DML 유입 영향을 봐야 합니다.

### 왜 상대적으로 유리한가

- 대량 row delete처럼 한 건씩 지우지 않아도 됨
- undo와 purge 부담이 row delete보다 훨씬 적을 수 있음
- 오래된 데이터 정리에 매우 유리함

### 그래도 주의해야 할 점

- partition 메타데이터 변경에는 metadata lock이 필요함
- 해당 테이블로 들어오는 DML이 많으면 lock 대기 구간이 체감될 수 있음
- partition 경계와 맞물린 쿼리가 순간적으로 영향을 받을 수 있음
- replication 환경에서는 replica에도 같은 DDL 영향이 전달됨

### 운영에서 체감하는 증상

- 아주 짧은 구간의 statement stall
- 특정 시간대에 drop과 겹친 write latency 튐
- replica apply 지연

### 실무 관점 요약

partition drop은 대량 delete보다 보통 훨씬 낫지만, "아무 영향 없이 사라지는 작업"은 아닙니다. 특히 busy table에서는 metadata lock 구간을 반드시 의식해야 합니다.

## 6. 세 작업을 비교해서 보면

| 작업 | DML 유입 시 주요 부담 | 운영상 체감 포인트 |
| --- | --- | --- |
| 인덱스 추가 | 백그라운드 인덱스 생성 + 동시 DML 반영 비용 | write latency, I/O, replica lag |
| 인덱스 삭제 | metadata 반영 + 실행 계획 변화 | query plan 악화, scan 증가, lock footprint 확대 |
| partition drop | metadata 변경 중심 | 짧은 stall, lock 대기, replica 지연 |

## 6-1. ADD COLUMN 시 DML 유입 영향

ADD COLUMN은 MySQL 버전과 컬럼 변경 형태에 따라 체감이 크게 달라집니다.

### 상대적으로 가벼운 경우

- `INSTANT`로 처리 가능한 단순 컬럼 추가
- 메타데이터 중심으로 끝나는 변경

이 경우에는 운영 체감이 작을 수 있지만, 그래도 시작과 끝의 metadata lock 구간은 의식해야 합니다.

### 무거워질 수 있는 경우

- `INPLACE` 또는 사실상 리빌드가 필요한 형태의 컬럼 추가
- default, row format, 제약조건, 저장 형식 때문에 내부 작업이 커지는 경우
- hot table에 컬럼 추가가 겹치는 경우

운영에서 느끼는 증상:

- 짧은 metadata lock 대기
- 쓰기 처리량 저하
- replica lag 증가
- 디스크/버퍼풀 압박

핵심은 "컬럼을 하나 더 붙이는 것"처럼 보여도 실제 알고리즘이 `INSTANT`가 아니면 상당히 비싼 작업이 될 수 있다는 점입니다.

## 6-2. MODIFY COLUMN 시 DML 유입 영향

MODIFY COLUMN은 운영상 가장 조심해야 하는 변경 중 하나입니다.

### 왜 더 위험한가

- 타입 변경, 길이 축소, NULL 속성 변경 등은 테이블 리빌드가 필요할 수 있음
- 리빌드가 필요하면 사실상 긴 백그라운드 작업과 더 큰 자원 사용량이 발생함
- live DML이 계속 들어오면 전체 비용이 더 커짐

### 운영에서 체감하는 문제

- ALTER TABLE이 오래 걸림
- CPU, I/O, temp 공간 사용량 증가
- write latency와 replica lag 악화
- 시작은 online처럼 보여도 실제로는 운영 영향이 크게 나타남

실무에서는 MODIFY COLUMN을 볼 때 먼저 "이게 metadata 수준 변경인가, 리빌드성 변경인가"를 구분해야 합니다.

## 6-3. PRIMARY KEY 변경 시 DML 유입 영향

PRIMARY KEY 변경은 보통 가장 무거운 축에 속합니다.

### 왜 비싼가

- InnoDB는 clustered index가 PRIMARY KEY 기반이므로, PRIMARY KEY를 추가하거나 바꾸는 작업은 테이블 구조 자체를 크게 흔듦
- 보조 인덱스에도 연쇄 영향이 갈 수 있음
- DML이 계속 들어오면 그 비용을 더 비싼 형태로 같이 부담하게 됨

### 운영에서 느끼는 증상

- ALTER 시간이 길어짐
- 디스크와 I/O 사용량이 크게 증가함
- replica가 매우 느리게 따라올 수 있음
- 바쁜 서비스에서는 사실상 위험 작업으로 분류해야 함

핵심 포인트:

- PRIMARY KEY 변경은 "온라인 옵션이 있으니 괜찮다"고 보기보다
- "테이블 핵심 구조를 바꾸는 고위험 변경"으로 보고 사전 검증과 점검 창을 더 강하게 잡는 편이 안전합니다.

## 7. 운영자가 특히 조심해야 할 패턴

### 1) 긴 트랜잭션이 이미 존재하는 상태

DDL보다 먼저 열린 트랜잭션이 metadata lock을 오래 잡고 있으면, 온라인 DDL도 시작 단계에서 막힐 수 있습니다.

### 2) batch DML과 DDL을 겹쳐서 실행하는 경우

대량 `UPDATE`/`DELETE` batch와 인덱스 작업을 동시에 돌리면 둘 다 서로를 더 비싸게 만들 수 있습니다.

### 3) primary만 보고 판단하는 경우

primary는 버텨도 replica가 뒤처질 수 있습니다. 특히 DDL과 heavy DML이 겹치면 replica apply가 더 힘들어집니다.

### 4) 인덱스 삭제 후 실행 계획 검증을 안 하는 경우

삭제 작업은 금방 끝났는데, 그 뒤부터 서비스가 서서히 느려지는 경우가 있습니다. 이건 계획 변화 때문에 생기는 전형적인 패턴입니다.

### 5) MODIFY COLUMN이나 PRIMARY KEY 변경을 가벼운 작업으로 오해하는 경우

이 두 작업은 종종 실제로는 리빌드성 비용을 동반합니다. 특히 live DML이 많을수록 예상보다 훨씬 무거운 작업이 됩니다.

### 6) partition drop 자체는 빠르다고 생각하고 사전 정리를 안 하는 경우

partition drop은 row delete보다 낫지만, 긴 트랜잭션과 metadata lock 대기가 있으면 busy hour에 충분히 체감 장애로 이어질 수 있습니다.

## 8. 안전한 운영 방법

### 작업 전

- 오래 열린 트랜잭션이 없는지 확인함
- 해당 테이블의 write traffic 수준을 확인함
- replica lag 여유가 있는지 확인함
- 인덱스 삭제라면 대체 실행 계획을 미리 점검함
- partition drop이라면 정말 파티션 전략이 delete보다 나은 상황인지 확인함
- ADD/MODIFY COLUMN이라면 실제 알고리즘이 `INSTANT`, `INPLACE`, 리빌드성 작업 중 무엇인지 먼저 확인함
- PRIMARY KEY 변경이라면 별도 점검 창과 fallback 전략을 더 엄격하게 준비함

### 작업 중

- metadata lock 대기를 관찰함
- write latency와 lock wait를 봄
- CPU, I/O, flush pressure를 봄
- replica lag를 함께 봄

### 작업 후

- 인덱스 추가 후 쿼리 계획이 기대대로 바뀌었는지 확인함
- 인덱스 삭제 후 scan 증가가 없는지 확인함
- partition drop 후 관련 배치나 조회 쿼리에 이상이 없는지 확인함
- replica가 정상적으로 따라잡는지 확인함
- ADD/MODIFY COLUMN 후 예상치 못한 rebuild 흔적이나 쿼리 계획 변화가 없는지 확인함
- PRIMARY KEY 변경 후 핵심 쿼리와 보조 인덱스 경로가 안정적인지 확인함

## 8-1. partition drop 영향 최소화 전략

사용자가 운영에서 가장 궁금해할 만한 부분만 따로 정리하면 아래와 같습니다.

### 1) 긴 트랜잭션을 먼저 정리하기

partition drop 자체보다 더 자주 문제를 만드는 것은 오래 열린 트랜잭션입니다.

- 작업 전에 long transaction을 먼저 확인함
- idle in transaction 세션이 없는지 봄
- metadata lock 대기 체인을 만들 수 있는 세션을 정리함

### 2) 피크 시간대를 피하기

partition drop은 대량 delete보다 훨씬 유리하지만, busy table에서는 짧은 stall도 체감됩니다.

- write peak 시간대는 피함
- 배치 대량 유입 시간과 겹치지 않게 함
- replica lag가 이미 높은 시간대는 피함

### 3) primary와 replica를 같이 본다

- primary만 보고 "빨리 끝났다"고 판단하지 않음
- replica apply 지연을 함께 봄
- stale read 민감 서비스면 lag threshold를 더 엄격하게 잡음

### 4) row delete 대신 partition strategy를 유지한다

partition drop의 가장 큰 장점은 row-by-row delete를 피하는 데 있습니다.

- 주기적 삭제가 필요하면 partition scheme을 계속 유지함
- 운영에서 오래된 데이터는 delete보다 partition drop이 가능한 구조로 설계함

### 5) 가능하면 더 짧은 메타데이터 작업으로 끝내기

partition drop은 본질적으로 metadata 성격이 강한 작업이므로, 불필요하게 다른 DDL과 묶지 않는 것이 좋습니다.

- drop partition만 단독으로 수행함
- 같은 배포에서 인덱스 변경과 함께 묶지 않음
- 관련 maintenance를 쪼개서 수행함

### 6) EXCHANGE PARTITION 패턴을 검토하기

트래픽이 매우 민감한 환경이라면 `EXCHANGE PARTITION` 패턴도 검토할 수 있습니다.

생각 방식은 이렇습니다.

- 먼저 partition을 별도 테이블과 교환함
- 사용자 트래픽 경로에서는 대상 데이터를 빨리 떼어냄
- 실제 삭제나 후속 정리는 별도 시점에 수행함

이 방식은 절차가 더 복잡하지만, 메인 테이블에 주는 체감 영향을 줄이는 데 도움이 될 수 있습니다.

### 7) 작업을 아주 단순하게 유지하기

실전에서는 복잡한 작업보다 단순한 작업이 안전합니다.

- 한 번에 하나의 partition만 처리함
- 작업 전후 지표 확인 구간을 둠
- 이상 징후가 보이면 다음 partition은 미룸

## 9. 빠른 체크리스트

DDL 전에:

- 긴 transaction 없음
- hot table 여부 확인
- replica 상태 확인
- 작업 목표와 rollback 전략 확인

DDL 중:

- metadata lock 대기 확인
- write latency 확인
- lock wait/deadlock 확인
- replica lag 확인

DDL 후:

- 쿼리 계획 확인
- 사용자 영향 확인
- replica recovery 확인

## 10. 한 줄 결론

- 인덱스 추가는 DML이 계속 들어오면 "가능하지만 비싸다"
- 인덱스 삭제는 작업보다도 "삭제 후 계획 변화"가 더 위험하다
- partition drop은 대량 delete보다 낫지만 metadata lock과 replication 영향은 여전히 본다

## 참고 자료

- MySQL Reference Manual, Online DDL Operations
- MySQL Reference Manual, Online DDL Performance and Concurrency
- MySQL Reference Manual, Online DDL Limitations
