# 스프링 배치 시작 가이드

## 최소 구성

처음 시작할 때는 아래 정도만 있으면 된다.

- Java
- Spring Boot
- Spring Batch
- 작업 메타데이터를 저장할 관계형 DB
- 실제 입력원 하나와 출력원 하나

보통 실무 시작점은 아래 그림에 가깝다.

```text
Spring Boot 앱
  + Spring Batch
  + DB(메타데이터 저장)
  + 입력원(CSV or DB)
  + 출력원(DB or 파일)
```

## 가장 작은 예시 시나리오

처음에는 `CSV 파일을 읽어 DB에 저장`하는 예제가 가장 이해하기 쉽다.

이 예제에서 필요한 것은 보통 다음과 같다.

1. Job 1개
2. Step 1개
3. Reader 1개
4. 필요하면 Processor 1개
5. Writer 1개

## 설계 순서 추천

### 1. 먼저 업무를 한 문장으로 정의

예:

- 매일 새벽 주문 정산 대상 데이터를 읽어 정산 테이블에 반영한다.
- 업로드된 회원 CSV를 읽어 정상 데이터만 적재한다.

### 2. 입력, 가공, 출력을 분리

- 어디서 읽는가
- 무엇을 검증하거나 변환하는가
- 어디에 저장하는가

### 3. 실패 정책을 먼저 정하기

- 한 건 오류면 전체 중단인가
- 일부 오류는 skip 가능한가
- 일시적 오류는 retry 할 것인가
- 재시작 시 어디서부터 이어갈 것인가

### 4. 운영 기준을 정하기

- 누가 실행하나
- 언제 실행하나
- 실패 알림은 어디로 가나
- 재처리 절차는 무엇인가

## 아주 작은 코드 골격

```java
@Configuration
public class BatchConfig {

    @Bean
    public Job importJob(JobRepository jobRepository, Step importStep) {
        return new JobBuilder("importJob", jobRepository)
                .start(importStep)
                .build();
    }

    @Bean
    public Step importStep(
            JobRepository jobRepository,
            PlatformTransactionManager transactionManager,
            ItemReader<InputRow> reader,
            ItemProcessor<InputRow, OutputRow> processor,
            ItemWriter<OutputRow> writer
    ) {
        return new StepBuilder("importStep", jobRepository)
                .<InputRow, OutputRow>chunk(100, transactionManager)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }
}
```

위 코드는 핵심 뼈대만 보여 준다.

- `Job`: 전체 작업
- `Step`: 실제 처리 단계
- `chunk(100, ...)`: 100건씩 묶어서 처리

## 처음 만들 때 추천 원칙

- 처음부터 병렬 처리하지 말고 단일 Step으로 시작
- skip, retry 정책은 업무 성격에 맞춰 최소한으로 설정
- 청크 크기는 추정하지 말고 실제 데이터로 확인
- 운영 로그와 실패 알림 경로를 초기에 정리
- 비즈니스 규칙은 Processor나 서비스로 분리

## 자주 하는 실수

- 스케줄러 역할까지 스프링 배치가 한다고 오해
- 작은 작업에도 지나치게 복잡한 배치 구조 도입
- 오류 정책 없이 성공 케이스만 구현
- 재실행 시 중복 적재 문제를 미리 막지 않음
- 개발 환경 예제를 운영 환경에 그대로 복붙

## 버전 메모

- 공식 문서 메인 라인은 현재 `6.0.3` 안정 버전이다.
- 다만 실제 프로젝트는 `Spring Boot`가 관리하는 호환 버전을 기준으로 잡는 편이 안전하다.
- 오래된 블로그 예제는 XML 설정이나 이전 API를 쓸 수 있으니 현재 공식 문서와 비교해야 한다.

## 한 줄 결론

처음에는 `CSV -> 가공 -> DB 저장` 같은 단순한 흐름으로 시작하고, 그다음에 재시작 정책과 운영 기준을 붙이는 방식이 가장 안전하다.
