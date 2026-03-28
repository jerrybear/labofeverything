# JVM Warmup 조사 문서 안내

## 이 폴더는 무엇을 담고 있나

이 폴더는 `JVM warmup issue`를 비전문가도 이해할 수 있게 정리한 조사 문서 묶음이다. 시작 직후 왜 느린지, 무엇이 실제 원인인지, 어떻게 측정해야 하는지, 운영에서 어떻게 완화할 수 있는지를 순서대로 읽을 수 있게 구성했다.

## 추천 읽기 순서

### 1. 가장 먼저 보기

- `01_overview.md`
- 목적: JVM 웜업 이슈를 한눈에 이해
- 다루는 내용: 정의, 대표 증상, 왜 생기는지, 언제 문제인지

### 2. 원인을 구조적으로 이해하고 싶다면

- `02_causes.md`
- 목적: JIT, 프로파일링, 클래스 로딩, GC, 캐시 관점에서 원인 이해
- 다루는 내용: tiered compilation, deoptimization, one-time initialization, warmup이 길어지는 이유

### 3. 제대로 측정하고 대응하고 싶다면

- `03_measurement-and-mitigation.md`
- 목적: startup, warmup, steady state를 분리해 보고 대응
- 다루는 내용: 흔한 측정 실수, 관측 포인트, 운영 대응, CDS/AppCDS, pre-warming, JMH

### 4. 근거를 확인하려면

- `10_sources.md`
- 목적: 공식 문서와 참고 자료 출처 확인
- 다루는 내용: Oracle JVM 문서, CDS 문서, OpenJDK JMH, 보조 참고 자료

### 5. 팀 공유용으로 빠르게 전달하려면

- `04_team-summary.md`
- 목적: 회의나 슬랙 공유용 1페이지 요약
- 다루는 내용: 핵심 메시지, 오해 방지 포인트, 바로 할 일

### 6. 직접 측정하거나 실험해 보려면

- `05_measurement-examples.md`
- 목적: 서비스 단위 측정 예시와 JMH 입문 예시 확인
- 다루는 내용: 측정 프레임, 부하 시나리오, JMH 샘플, 해석 주의점

### 7. Spring Boot 기준으로 바로 대응하려면

- `06_spring-boot-playbook.md`
- 목적: Spring Boot 서비스에서 실제로 적용할 대응 순서 정리
- 다루는 내용: pre-warming, readiness, 초기화 정리, 운영 체크리스트

### 8. 운영 대시보드 항목을 바로 잡고 싶다면

- `07_observability-dashboard.md`
- 목적: JVM warmup 관측용 대시보드와 알림 항목 정리
- 다루는 내용: 패널 목록, 지표 정의, 알림 기준, 해석 포인트

### 9. Spring Boot 대응 절차를 더 구체적으로 보려면

- `08_spring-boot-action-guide.md`
- 목적: 대응 순서와 예열 아이디어를 실행 단위로 더 구체화
- 다루는 내용: 단계별 액션, warmup 대상 선정법, 구현 패턴, 주의점

### 10. 예열 호출 횟수와 warmup 종류 차이를 알고 싶다면

- `09_warmup-call-strategy.md`
- 목적: 몇 번 호출해야 하는지와 초기화 예열 vs JIT 예열 차이 정리
- 다루는 내용: 호출 횟수 기준, 일반 개발 판단 기준, JVM 옵션 한계

## 문서 간 관계

```text
01_overview
  -> 02_causes
  -> 03_measurement-and-mitigation
  -> 04_team-summary
  -> 05_measurement-examples
  -> 06_spring-boot-playbook
  -> 07_observability-dashboard
  -> 08_spring-boot-action-guide
  -> 09_warmup-call-strategy
  -> 10_sources
```

- `01_overview.md`는 전체 지형을 잡아 주는 입문 문서다.
- `02_causes.md`는 실제로 왜 startup 직후 성능이 달라지는지 설명한다.
- `03_measurement-and-mitigation.md`는 측정과 대응 방법을 실무 관점에서 정리한다.
- `04_team-summary.md`는 팀 공유용으로 바로 전달할 수 있는 압축판이다.
- `05_measurement-examples.md`는 실제 측정과 실험 출발점을 제공한다.
- `06_spring-boot-playbook.md`는 Spring Boot 환경의 실전 대응 순서를 정리한다.
- `07_observability-dashboard.md`는 운영 대시보드와 알림 기준을 정리한다.
- `08_spring-boot-action-guide.md`는 Spring Boot 대응 절차를 더 실행 가능하게 풀어 쓴 확장판이다.
- `09_warmup-call-strategy.md`는 예열 호출 횟수와 warmup 종류를 구분해 설명한다.
- `10_sources.md`는 위 문서들의 근거를 모아 둔 문서다.

## 빠른 선택 가이드

- `왜 처음만 느린지` 알고 싶으면 `01_overview.md`
- `JIT와 warmup 관계`가 궁금하면 `02_causes.md`
- `벤치마크를 어떻게 봐야 하는지` 궁금하면 `03_measurement-and-mitigation.md`
- `팀에 짧게 설명해야 하면` `04_team-summary.md`
- `직접 실험을 시작하려면` `05_measurement-examples.md`
- `Spring Boot에서 바로 적용하려면` `06_spring-boot-playbook.md`
- `운영 대시보드로 보고 싶으면` `07_observability-dashboard.md`
- `Spring Boot 대응 절차를 더 자세히 보려면` `08_spring-boot-action-guide.md`
- `예열 호출 횟수가 궁금하면` `09_warmup-call-strategy.md`
- `근거 검증`이 필요하면 `10_sources.md`

## 한 줄 요약

이 폴더는 `입문 이해 -> 원인 구조화 -> 측정/대응 -> 출처 검증` 순으로 읽도록 설계한 JVM warmup 조사 묶음이다.
