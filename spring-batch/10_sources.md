# 스프링 배치 조사 출처

## 우선 참고한 공식 문서

1. Spring Batch Reference Documentation
   - URL: `https://docs.spring.io/spring-batch/reference/`
   - 확인 포인트: 개요, 아키텍처, 도메인 개념, Step 구성, 재시작, skip, retry, 통합, 관측성

2. Spring Batch Overview
   - URL: `https://docs.spring.io/spring-batch/reference/index.html`
   - 확인 포인트: 현재 문서 구조와 안정 버전 라인

3. Spring Batch Introduction
   - URL: `https://docs.spring.io/spring-batch/reference/spring-batch-intro.html`
   - 확인 포인트: 배치 처리 정의, 사용 시나리오, 기본 방향

4. The Domain Language of Batch
   - URL: `https://docs.spring.io/spring-batch/reference/domain.html`
   - 확인 포인트: Job, Step, JobInstance, JobExecution, ExecutionContext

5. Chunk-oriented Processing / Restart / Skip / Retry
   - URL: `https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing.html`
   - URL: `https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/restart.html`
   - URL: `https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html`
   - URL: `https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/retry-logic.html`
   - 확인 포인트: 청크 처리, 재시작, 예외 정책

6. TaskletStep
   - URL: `https://docs.spring.io/spring-batch/reference/step/tasklet.html`
   - 확인 포인트: Tasklet 기반 작업 구조

7. What’s New in Spring Batch 6
   - URL: `https://docs.spring.io/spring-batch/reference/whatsnew.html`
   - 확인 포인트: 최신 변경 사항, 버전 민감 포인트

8. Spring Framework Scheduling
   - URL: `https://docs.spring.io/spring-framework/reference/integration/scheduling.html`
   - 확인 포인트: `@Scheduled`, 스케줄링 추상화, 기본 사용 범위

9. Quartz Scheduler
   - URL: `https://www.quartz-scheduler.org/`
   - 확인 포인트: 스케줄링 역할, 트리거, 영속 스케줄링, 클러스터링 관점

## 조사 메모

- 공식 문서 메인 페이지에서 안정 버전 라인으로 `6.0.3`이 표시되는 것을 확인했다.
- 다만 실제 적용 버전은 프로젝트의 `Spring Boot` 및 주변 스택과의 호환성을 함께 확인해야 한다.
- 실무 입문 관점에서는 공식 문서를 우선 기준으로 보고, 오래된 블로그 예제는 API 차이를 반드시 대조하는 것이 안전하다.
- 스케줄링 비교 정리는 `Spring Batch는 처리`, `Quartz/@Scheduled는 실행 시점 제어`라는 책임 구분을 중심으로 재구성했다.

## 이번 정리의 관점

- 비전문가도 이해할 수 있는 설명을 우선했다.
- 실무 판단에 필요한 `언제 쓰는지`, `왜 쓰는지`, `어디까지가 역할인지`를 중심으로 재구성했다.
- 코드 자체보다 운영 개념과 구조 이해를 먼저 잡는 데 초점을 맞췄다.
