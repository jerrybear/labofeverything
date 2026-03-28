# JVM Warmup 측정 예시와 JMH 시작점

## 이 문서의 목적

이 문서는 `JVM warmup이 있는지`를 실제로 확인할 때 어떤 식으로 측정 프레임을 잡아야 하는지, 그리고 작은 코드 단위 비교에는 왜 `JMH`를 쓰는지 예시 중심으로 정리한 문서다.

## 먼저 구분할 두 가지

### 서비스 단위 측정

실제 API, 배치, 워커가 `언제 준비되고 언제 안정화되는지`를 보는 측정이다.

### 코드 단위 측정

특정 메서드나 알고리즘이 더 빠른지 비교하는 측정이다.

서비스 단위는 부하 테스트 도구와 관측 지표가 중요하고, 코드 단위는 JMH가 더 적합하다.

## 서비스 단위 측정 예시

### 목적

- readiness까지 몇 초 걸리는지
- 첫 1분 p95/p99 latency가 어떤지
- 몇 분 뒤 처리량이 얼마나 안정되는지
- 새 인스턴스 추가 시 tail latency가 흔들리는지

### 예시 프레임

1. 완전히 차가운 인스턴스를 하나 띄운다.
2. readiness 통과 시점을 기록한다.
3. 실제와 비슷한 요청을 5~10분 동안 지속적으로 보낸다.
4. 10초 또는 30초 단위로 latency, throughput, CPU를 본다.
5. 첫 1분과 이후 5분을 나눠 비교한다.

### 꼭 같이 볼 지표

- readiness 시간
- p50, p95, p99 latency
- requests per second
- CPU 사용률
- GC pause와 allocation rate
- 새 인스턴스가 붙는 시점의 tail latency

### 해석 예시

- readiness는 12초인데 첫 60초 p99가 높으면: `기동은 됐지만 warmup은 안 끝남`
- 평균 latency는 비슷한데 p99만 높으면: `초반 tail latency 문제 가능성`
- CPU spike와 함께 처리량이 점진적으로 좋아지면: `JIT warmup 가능성`
- 첫 요청만 느리고 이후는 빠르면: `초기화/캐시 미적재 영향 가능성`

## 간단한 서비스 관측 기록 템플릿

```text
Environment:
- JDK:
- Container CPU/Memory:
- Instance type:
- Traffic pattern:

Phase metrics:
- Time to readiness:
- p95 latency during first 1 minute:
- p99 latency during first 1 minute:
- Stable throughput reached at:
- Peak startup CPU:

Likely causes:
- JIT / class loading / cache / GC / unknown

Next action:
- pre-warming / min instances / CDS / initialization cleanup / more profiling
```

## JMH는 언제 쓰나

JMH는 특정 코드 조각의 성능 비교를 더 믿을 만하게 하기 위한 JVM 표준 벤치마크 도구다.

- 작은 메서드 비교
- 자료구조 비교
- 직렬화/파싱 코드 비교
- 알고리즘 변경 전후 비교

하지만 `서비스 전체 startup 문제`를 그대로 대변하지는 않는다.

## JMH 예시 코드

```java
import java.util.concurrent.TimeUnit;
import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.BenchmarkMode;
import org.openjdk.jmh.annotations.Measurement;
import org.openjdk.jmh.annotations.Mode;
import org.openjdk.jmh.annotations.OutputTimeUnit;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.State;
import org.openjdk.jmh.annotations.Warmup;

@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
public class StringConcatBenchmark {

    private final String a = "hello";
    private final String b = "world";

    @Benchmark
    @Warmup(iterations = 5)
    @Measurement(iterations = 10)
    public String plusOperator() {
        return a + b;
    }

    @Benchmark
    @Warmup(iterations = 5)
    @Measurement(iterations = 10)
    public String builder() {
        return new StringBuilder().append(a).append(b).toString();
    }
}
```

## 이 예시에서 중요한 점

- `@Warmup`과 `@Measurement`를 분리한다.
- 한두 번 측정한 숫자를 바로 믿지 않는다.
- JMH가 JVM 최적화 함정을 줄여 주지만, 실제 서비스 환경을 완전히 대신하지는 않는다.

## 실험 설계 팁

### 서비스 warmup 확인용

- cold start와 warmed start를 나눠 기록한다.
- 새 인스턴스 1개 추가 상황을 따로 본다.
- pre-warming 적용 전후를 비교한다.

### 코드 비교용

- JMH로 마이크로벤치마크를 만든다.
- 입력 크기와 데이터 분포를 현실적으로 만든다.
- 결과는 평균뿐 아니라 분산과 반복 일관성도 본다.

## 한 줄 결론

서비스의 JVM warmup 문제를 보려면 `시간 흐름에 따른 시스템 지표`를 보고, 작은 코드 비교를 하려면 `JMH 같은 전용 도구`를 쓰는 식으로 측정 대상을 분리해야 한다.
