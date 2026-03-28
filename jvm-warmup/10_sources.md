# JVM Warmup 조사 출처

## 우선 참고한 공식 문서

1. Java HotSpot Virtual Machine Performance Enhancements
   - URL: `https://docs.oracle.com/en/java/javase/24/vm/java-hotspot-virtual-machine-performance-enhancements.html`
   - 확인 포인트: tiered compilation, segmented code cache, HotSpot 최적화 구조

2. Class Data Sharing
   - URL: `https://docs.oracle.com/en/java/javase/22/vm/class-data-sharing.html`
   - 확인 포인트: CDS/AppCDS 개념, startup 시간 단축, shared archive 동작

3. OpenJDK JMH
   - URL: `https://openjdk.org/projects/code-tools/jmh/`
   - 확인 포인트: JVM 마이크로벤치마크 표준 도구 위치와 성격

## 보조 참고 자료

4. How to Warm Up the JVM
   - URL: `https://www.baeldung.com/java-jvm-warmup`
   - 확인 포인트: warmup 개념 설명, 예시 중심 입문 자료

5. Inside Java: AOT cache related article
   - URL: `https://inside.java/2026/01/09/run-aot-cache/`
   - 확인 포인트: 최신 JDK 계열에서 cold start 개선 흐름 참고

## 조사 메모

- 공식 문서상 HotSpot은 tiered compilation을 기본 성능 향상 요소로 설명한다.
- 공식 문서상 CDS는 startup time과 memory footprint를 줄이는 기능으로 설명된다.
- 따라서 warmup 논의에서는 `JIT 최적화 구간`과 `class loading/startup 최적화 구간`을 분리해 보는 것이 중요하다.
- JMH는 서비스 전체의 startup 문제를 직접 해결하는 도구가 아니라, JVM 코드 측정 실수를 줄이기 위한 벤치마크 도구로 보는 것이 맞다.

## 이번 정리의 관점

- 비전문가도 이해할 수 있는 설명을 우선했다.
- 실무 판단에 필요한 `왜 처음이 느린지`, `무엇을 분리 측정해야 하는지`, `어떤 환경에서 더 아픈지`를 중심으로 재구성했다.
- JVM 옵션 나열보다 현상 이해와 운영 판단 기준을 먼저 잡는 데 초점을 맞췄다.
