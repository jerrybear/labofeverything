# Native Online DDL vs pt-online-schema-change

## 이 문서는 무엇을 설명하나

이 문서는 MySQL의 native online DDL과 `pt-online-schema-change`를 운영 관점에서 비교합니다.

비교 초점은 다음입니다.

- lock 동작
- live DML에 주는 부담
- trigger 영향
- replication 영향
- foreign key 이슈
- 운영 복잡도
- 어떤 상황에서 무엇이 더 안전한지

## 먼저 한 줄로 요약하면

- native online DDL은 단순하고 MySQL 기본 기능에 가깝지만, metadata lock과 알고리즘 제약을 잘 이해해야 합니다.
- `pt-online-schema-change`는 더 우회적인 방식으로 위험한 변경을 다룰 수 있지만, trigger 기반이어서 DML 오버헤드와 운영 복잡도가 커집니다.
- 둘 중 무엇이 더 좋은가는 "기능"보다 "현재 테이블 크기, DML 유입량, 복제 구조, foreign key, trigger 존재 여부"에 따라 달라집니다.

## 1. 동작 방식 차이

### Native online DDL

기본적으로 MySQL 엔진 안에서 `ALTER TABLE`을 직접 수행합니다.

- `INSTANT`, `INPLACE`, `COPY` 같은 알고리즘 차이가 있음
- online처럼 보여도 metadata lock 구간은 존재함
- 변경 종류에 따라 rebuild 여부가 크게 달라짐

### pt-online-schema-change

기본 아이디어는 "새 테이블을 만들고 조금씩 복사한 뒤 마지막에 스왑"입니다.

- 원본 테이블과 동일한 새 테이블 생성
- row를 chunk 단위로 복사
- 원본 테이블에 trigger를 걸어 변경분 동기화
- 마지막 순간 테이블 이름을 바꾸며 전환

즉, native는 엔진 내부 변경이고, `pt-online-schema-change`는 우회 복사 전략에 가깝습니다.

## 2. lock 관점 비교

### Native online DDL의 특징

- 시작과 끝에 metadata lock이 필요함
- 작업 종류에 따라 동시 DML 허용 범위가 달라짐
- 잘 맞는 작업은 아주 매끄럽게 끝나지만, 안 맞는 작업은 대기 체인을 만들 수 있음

### pt-online-schema-change의 특징

- 긴 시간 동안 원본 테이블을 강하게 막지는 않음
- 대신 마지막 스왑 시점의 lock을 조심해야 함
- 장시간 복사 동안 시스템은 계속 추가 부하를 감당해야 함

운영상 체감은 이렇습니다.

- native: 짧지만 민감한 락 구간
- pt-osc: 길게 이어지는 부하와 마지막 전환 구간의 주의점

## 3. live DML에 주는 부담

### Native online DDL

- 작업 종류에 따라 상대적으로 오버헤드가 작을 수 있음
- 하지만 인덱스 생성, rebuild성 변경, PK 변경은 여전히 무거움
- DML이 많으면 내부 변경 추적과 replication 비용이 올라감

### pt-online-schema-change

- 원본 테이블로 들어오는 `INSERT`/`UPDATE`/`DELETE`마다 trigger가 작동함
- 즉, DML이 많을수록 추가 오버헤드가 직접 커짐
- busy table에서는 write latency가 더 눈에 띄게 악화될 수 있음

핵심 차이:

- native는 엔진 내부 비용 중심
- pt-osc는 trigger와 row copy에 따른 외부화된 비용 중심

## 4. trigger 영향

### Native online DDL

- 별도 trigger가 필요 없음
- 기존 trigger 구조와 충돌 가능성이 낮음

### pt-online-schema-change

- 자체 trigger를 사용함
- 이미 trigger가 있는 테이블에서는 적용 제약이 생길 수 있음
- trigger 자체가 write 경로를 더 무겁게 만들 수 있음

운영상 의미:

- trigger가 많은 시스템, 로직이 복잡한 시스템에서는 pt-osc의 부담을 더 신중하게 봐야 함

## 5. replication 영향

### Native online DDL

- DDL 자체가 복제됨
- 변경 종류에 따라 replica가 같은 부담을 다시 겪을 수 있음
- primary는 버텨도 replica apply나 DDL 재실행에서 지연이 날 수 있음

### pt-online-schema-change

- row copy와 trigger 기반 변경이 replication에 더 많은 부담을 줄 수 있음
- chunk 복사 동안 replica lag가 누적될 수 있음
- lag threshold 제어를 반드시 같이 봐야 함

운영상 느낌은 보통 이렇습니다.

- native는 "짧지만 강한 영향"
- pt-osc는 "길고 누적되는 영향"

## 6. foreign key 관점

### Native online DDL

- MySQL 자체 제약 안에서 비교적 자연스럽게 동작함
- 물론 작업 종류에 따라 여전히 제약은 존재함

### pt-online-schema-change

- foreign key가 있으면 운영 난도가 확 올라감
- rename/swap 과정과 참조 관계를 더 신중하게 다뤄야 함
- cascading 관계가 복잡하면 위험도가 커짐

실무에서는 foreign key가 복잡한 테이블일수록 pt-osc를 쉽게 선택하지 않는 경우가 많습니다.

## 7. 운영 복잡도 비교

### Native online DDL

- SQL 한 문장으로 처리 가능
- MySQL 버전별 알고리즘만 잘 이해하면 됨
- 단순하고 기본 기능 중심

### pt-online-schema-change

- 도구 설치 필요
- 옵션 이해 필요
- chunk size, lag threshold, critical load 같은 운영 파라미터를 잡아야 함
- dry-run, cutover, fallback 절차까지 운영자가 더 많이 챙겨야 함

즉, pt-osc는 강력하지만 훨씬 더 운영형 도구입니다.

## 8. 언제 native가 더 안전한가

다음 상황에서는 native online DDL이 더 자연스러운 선택인 경우가 많습니다.

- `INSTANT`나 가벼운 `INPLACE`로 끝나는 변경
- trigger가 이미 있는 테이블
- foreign key 구조가 복잡한 테이블
- 작업이 단순하고 빠르게 끝날 가능성이 높은 경우
- 운영 복잡도를 최소화하고 싶은 경우

## 9. 언제 pt-online-schema-change가 더 안전할 수 있나

다음 상황에서는 `pt-online-schema-change`가 대안이 될 수 있습니다.

- native로는 사실상 rebuild성 고비용 작업인 경우
- 대형 테이블에서 더 세밀한 throttling이 필요한 경우
- cutover 전까지 원본 테이블을 유지하며 통제하고 싶은 경우
- chunk 기반으로 부하를 조절하며 천천히 진행하고 싶은 경우

단, 이것은 "더 쉬움"이 아니라 "더 복잡하지만 더 통제 가능할 수 있음"에 가깝습니다.

## 10. 빠른 판단 기준

| 상황 | 더 먼저 검토할 선택 |
| --- | --- |
| 단순 ADD COLUMN, 가벼운 인덱스 작업 | native online DDL |
| PK 변경, 무거운 MODIFY COLUMN | native 가능 여부 확인 후 pt-osc 검토 |
| trigger 이미 존재 | native 우선 |
| foreign key 복잡 | native 우선, pt-osc는 매우 신중 |
| busy table에서 세밀한 속도 조절 필요 | pt-osc 검토 |

## 11. 운영 체크리스트

어떤 방식을 택하든 공통적으로:

- long transaction 확인
- metadata lock 또는 cutover 영향 확인
- replica lag 기준 설정
- rollback 또는 중단 전략 정의
- 핵심 쿼리 계획 검증

pt-osc를 쓸 때 추가로:

- trigger 충돌 가능성 확인
- foreign key 영향 확인
- chunk, lag, load 파라미터 사전 검토
- cutover 시점 모니터링 강화

## 12. 한 줄 결론

- 단순하고 기본 기능에 잘 맞는 변경이면 native online DDL이 우선
- 더 무겁고 통제가 필요한 변경이면 pt-osc가 대안
- 다만 pt-osc는 "안전 자동화"가 아니라 "운영 난도를 받아들이는 대신 제어력을 얻는 선택"에 가깝습니다

## 참고 자료

- MySQL Reference Manual, Online DDL Operations
- MySQL Reference Manual, Online DDL Performance and Concurrency
- Percona Toolkit Documentation, pt-online-schema-change
