# [기술 블로그] Kafka 컨슈머 성능 최적화 분투기: 분당 1억건 이상 처리하기

## 프롤로그: 성능 한계에 부딪힌 데이터 증폭기

대용량 시계열 데이터(Time-Series Data)를 생성하여 TSDB에 쓰는 Kafka Consumer 애플리케이션(`tsdb-writer`)의 성능 검증(PoC)을 진행했습니다. PoC의 목표는 Kafka의 1분 단위 원본 데이터를 증폭시켜, 대규모 트래픽 상황에서 시스템의 한계를 파악하는 것이었습니다.

하지만 초기 테스트 결과는 암울했습니다. 데이터 증폭 배수(`M`)를 높일수록 효율이 급격히 감소하며 분당 약 1.4억 개의 처리량 한계(Throughput Ceiling)에 부딪혔습니다. 특히 `M=12` 설정에서는 효율이 **50%** 까지 떨어졌습니다.

| Test | Parallelism | M (증폭 배수) | 예상 처리량/분 | 실제 처리량/분 | 효율 |
|------|-------------|---|----------|---------|------------|
| 4 | 6 | 12 (5초) | 260.4M | ~130M | 50% |

이대로는 안 됐습니다. `tsdb-writer` 내부의 병목을 찾아내고, 시스템의 잠재력을 최대한 끌어올리기 위한 최적화 여정을 시작했습니다.

## 1단계: `tsdb-writer` 내부 병목 제거

초기 분석 결과, 병목은 `tsdb-writer` 애플리케이션 내부에 있었습니다. 두 가지 주요 개선을 진행했습니다.

### 개선 #1: 비동기 I/O 도입 (with Virtual Threads)

**진단:** 가장 큰 문제는 DB 쓰기(I/O) 작업이 Kafka 메시지를 처리하는 스레드를 막고 있다는 점이었습니다. 스레드는 데이터를 DB로 보내고 응답이 올 때까지 아무 일도 하지 않고 대기(Blocking)했고, 이 시간 동안 Kafka에서는 새로운 데이터를 가져오지 못해 전체 처리량이 저하되었습니다.

**변경 전: 동기식 아키텍처**

하나의 스레드가 모든 일을 순차적으로 처리하여, DB 쓰기(I/O) 작업이 전체 흐름을 막고 있었습니다.

```mermaid
graph TD
    subgraph SegmentConsumer Thread
        A[Kafka Poll] --> B{데이터 증폭}; 
        B --> C[timeSeriesService.write()];
        C -- I/O 대기 발생 (수초 소요) --> D[쓰기 완료];
        D --> A;
    end
```

<details>
<summary><b>변경 전 `SegmentConsumer.run()` 전체 코드 보기</b></summary>

```java
// SegmentConsumer.java (Before)
@Override
public void run() {
    Long currentOffset = initialOffset;
    try {
        SegmentDto currentSegment = getFirstSegment(initialOffset, parallelIdx);
        consumer.seek(topicPartition, currentSegment.getStartOffset());

        int retryCount = writeRetryCount;
        while (!Thread.currentThread().isInterrupted()) {
            List<ConsumerRecord<String, String>> records = consumer.poll(Duration.ofMillis(500)).records(topicPartition);

            if (records.isEmpty()) continue;

            List<TimeSeries> timeSeries = extractTimeSeries(records, currentSegment);
            try {
                // 이 부분에서 I/O 대기가 발생하여 전체 스레드가 멈춤
                if(!timeSeries.isEmpty()) timeSeriesService.write(storageTier, timeSeries);
            } catch (Exception e) {
                log.error("[TimeSeries Writer] Fail to write {} time series to DB", timeSeries.size(), e);
                consumer.seek(topicPartition, currentSegment.getStartOffset());
                retryCount--;
                if(retryCount >= 0) continue;
            }
            retryCount = writeRetryCount;

            if (records.getLast().offset() >= currentSegment.getEndOffset()) {
                log.info("[TimeSeries Writer] polled records' last offset: {}, current segment: {}", records.getLast().offset(), currentSegment);
                consumer.commitSync(Map.of(topicPartition, new OffsetAndMetadata(currentSegment.getEndOffset())), Duration.ofMillis(2000L));
                currentOffset = currentSegment.getEndOffset();

                currentSegment = getNextSegment(currentSegment);
                while (true) {
                    long logEndOffset = consumer.endOffsets(List.of(topicPartition)).get(topicPartition);
                    if(logEndOffset >= currentSegment.getStartOffset() || Thread.currentThread().isInterrupted()) break;
                    Thread.sleep(1000L);
                }

                consumer.seek(topicPartition, currentSegment.getStartOffset());
            }
        }
    } catch (Exception e) {
        log.error("[TimeSeries Writer] exception is thrown, {}", e.getMessage(), e);
    } finally {
        log.info("[TimeSeries Writer] commit current offset({})", currentOffset);
        consumer.commitSync(Map.of(topicPartition, new OffsetAndMetadata(currentOffset)), Duration.ofMillis(2000L));
        Thread.currentThread().interrupt();
    }
}
```
</details>

**해결:** Java 21의 가상 스레드(Virtual Thread)를 사용하여 쓰기 작업을 비동기로 전환했습니다. 이제 메인 스레드는 쓰기 작업을 가상 스레드 실행기에게 위임하고, 즉시 다음 데이터를 처리하러 갑니다.

**변경 후: 비동기 아키텍처**

```mermaid
graph TD
    subgraph SegmentConsumer Thread
        A[Kafka Poll] --> B{데이터 증폭};
        B --> Q{쓰기 작업 요청 (대기 없음)};
        Q --> A;
    end

    subgraph Virtual Thread Executor
        Q --> VT1[Virtual Thread 1: Write Batch 1];
        Q --> VT2[Virtual Thread 2: Write Batch 2];
        Q --> VTx[...];
    end
```

<details>
<summary><b>변경 후 `SegmentConsumer.run()` 전체 코드 보기</b></summary>

```java
// SegmentConsumer.java (After Async Write)
@Override
public void run() {
    Long currentOffset = initialOffset;
    try {
        SegmentDto currentSegment = getFirstSegment(initialOffset, parallelIdx);
        consumer.seek(topicPartition, currentSegment.getStartOffset());

        while (!Thread.currentThread().isInterrupted()) {
            List<ConsumerRecord<String, String>> records = consumer.poll(Duration.ofMillis(500)).records(topicPartition);

            if (records.isEmpty()) continue;

            List<TimeSeries> timeSeries = extractTimeSeries(records, currentSegment);

            if (!timeSeries.isEmpty()) {
                // 쓰기 작업을 가상 스레드 실행기에 위임하고, 자신은 즉시 다음 작업을 위해 복귀
                writerExecutor.submit(() -> {
                    int retries = writeRetryCount;
                    while (retries >= 0) {
                        try {
                            timeSeriesService.write(storageTier, timeSeries);
                            writtenCounter.addAndGet(timeSeries.size()); // 성공 시 카운트
                            break; // 성공
                        } catch (Exception e) {
                            retries--;
                            log.error("[WriterTask] Failed to write batch of size {}. Retries left: {}.", timeSeries.size(), retries, e);
                            if (retries < 0) {
                                log.error("[WriterTask] Exhausted retries for batch. Data will be lost.");
                            } else {
                                try {
                                    Thread.sleep(1000);
                                } catch (InterruptedException interruptedException) {
                                    Thread.currentThread().interrupt();
                                    break;
                                }
                            }
                        }
                    }
                });
            }

            if (records.getLast().offset() >= currentSegment.getEndOffset()) {
                // ... 후략 ...
            }
        }
    } catch (Exception e) { ... }
}
```
</details>

### 개선 #2: 메모리 최적화 (객체 재사용)

**진단:** 비동기 전환 후에도 높은 `M`값에서 CPU 사용률이 과도하게 높았습니다. 원인은 데이터 증폭 시 매번 `HashMap` 객체를 대량으로 생성하여 GC(Garbage Collection) 부하가 극심했기 때문입니다.

**해결:** `instanceNo`가 다른 `dimensions` Map들을 미리 한 번만 생성하여, 루프 안에서는 이것을 재사용하도록 로직을 변경했습니다. `M*N`번 생성되던 `HashMap`이 `N`번만 생성되도록 하여 메모리 할당량을 획기적으로 줄였습니다.

<details>
<summary><b>메모리 최적화 전/후 `expandTimeSeries` 전체 코드 보기</b></summary>

**변경 전 `expandTimeSeries` 메소드:**
```java
// PocTimeSeriesTransformer.java (Before)
private List<TimeSeries> expandTimeSeries(TimeSeries original) {
    // ...
    for (int i = 0; i < timeMultiplier; i++) {
        // ...
        long newTimestamp = baseTimestamp + (timeOffset * 1000L);

        for (int instanceOffset = 0; instanceOffset < instanceMultiplier; instanceOffset++) {
            TimeSeries expanded;
            if (instanceOffset == 0) {
                expanded = original.copyWithTimestamp(newTimestamp);
            } else {
                // 매 루프마다 새로운 HashMap 생성 및 복사 발생
                String originalInstanceNo = getInstanceNoFromDimensions(original);
                String newInstanceNo = generateNewInstanceNo(originalInstanceNo, instanceOffset);
                expanded = original.copyWithTimestampAndInstanceNo(newTimestamp, newInstanceNo);
            }
            result.add(expanded);
        }
    }
    return result;
}
```

**변경 후 `expandTimeSeries` 메소드:**
```java
// PocTimeSeriesTransformer.java (After)
private List<TimeSeries> expandTimeSeries(TimeSeries original) {
    // 1. 필요한 모든 dimensions Map 변형을 미리 생성 (N번만 생성)
    List<Map<String, String>> dimensionVariants = new ArrayList<>(instanceMultiplier);
    if (original.getDimensions() != null) {
        Map<String, String> baseDimensions = new HashMap<>(original.getDimensions());
        String originalInstanceNo = baseDimensions.remove("instanceNo");
        if (originalInstanceNo == null) originalInstanceNo = "unknown";

        for (int instanceOffset = 0; instanceOffset < instanceMultiplier; instanceOffset++) {
            Map<String, String> variant = new HashMap<>(baseDimensions);
            if (instanceOffset == 0) {
                variant.put("instanceNo", originalInstanceNo);
            } else {
                variant.put("instanceNo", generateNewInstanceNo(originalInstanceNo, instanceOffset));
            }
            dimensionVariants.add(variant);
        }
    }

    // ...
    // 2. 메인 루프(M번 반복)에서는 미리 생성된 Map을 재사용
    for (int i = 0; i < timeMultiplier; i++) {
        // ...
        for (Map<String, String> dims : dimensionVariants) {
            TimeSeries expanded = TimeSeries.builder()
                    .timestamp(newTimestamp)
                    .dimensions(dims) // HashMap을 새로 만들지 않고 재사용
                    .value(original.getValue())
                    .productKey(original.getProductKey())
                    // ...
                    .build();
            result.add(expanded);
        }
    }
    return result;
}
```
</details>

### 1단계 최적화 결과

-   `M=6` 설정에서 처리 효율이 기존 56%에서 **90%** 가까이 상승하며, **분당 약 1.2억 개**의 안정적인 처리량을 확보했습니다.
-   `tsdb-writer` 내부의 주요 병목이 해결되었음을 확인했습니다.

## 3. 2단계: 새로운 병목의 발견과 최종 원인 규명

최적화된 애플리케이션의 한계를 확인하기 위해 `M=12`로 부하를 높이자, 오히려 성능이 분당 1억 개 수준으로 **감소**하고 `Slow write request` 경고가 발생했습니다. 다양한 실험 끝에 **VictoriaMetrics 대시보드**를 직접 분석하여 다음과 같은 결정적인 증거를 확보했습니다.

1.  **`vminsert` CPU 사용률:** **~30%** 수준으로 매우 여유가 있었습니다.
2.  **`Concurrent inserts` (동시 쓰기 처리):** `tsdb-writer`가 200개에 가까운 연결을 맺고 요청을 보내고 있음에도 불구, `vminsert`는 **동시에 최대 16개의 쓰기 작업만 처리**하고 있었습니다.

**대시보드 스크린샷:**

@스크린샷 2025-09-23 오후 3.24.55.png
@스크린샷 2025-09-23 오후 4.39.11.png

**최종 원인:**
병목의 진짜 원인은 **`vminsert` 컴포넌트가 스스로를 보호하기 위해 의도적으로 설정한 '내부 동시 처리량 제한(Concurrency Limit)'** 이었습니다. `tsdb-writer`가 보낸 수많은 병렬 요청들은 `vminsert`의 내부 큐에서 자신의 차례(16개 중 하나의 슬롯)를 기다리고 있었고, 이 대기 시간이 바로 7~9초의 'Slow write request'의 정체였습니다.

## 4. 최종 결론 및 향후 전략

-   **`tsdb-writer` 최적화 완료:** `tsdb-writer` 애플리케이션은 외부 시스템이 병목이 될 때까지 성공적으로 최적화되었습니다.
-   **단일 인스턴스 최적 운영점:** `time-multiplier=6`으로 **분당 약 1.2억 개**를 처리하는 것이 가장 효율적입니다.
-   **확장 전략:** 이 이상의 처리량이 필요할 경우, 유일하고 올바른 방법은 **'시스템 전반의 수평 확장'** 입니다. 즉, `M=6`으로 설정된 `tsdb-writer` 인스턴스를 추가하는 동시에, 늘어난 부하를 감당할 수 있도록 **`vminsert` 노드 또한 증설하고 부하를 분산**해야 합니다.

---

### 부록: `vminsert` 동시성 제한(16개) 사실 검증

VictoriaMetrics 공식 자료를 통해, `vminsert`의 동시 처리 제한이 기본적으로 **`CPU 코어 수 * 2`** 로 설정되는 것을 확인했습니다. 대시보드에서 `vminsert` 서버의 CPU 코어가 **8개**인 것을 확인했으며, 이는 `8 * 2 = 16`이라는 계산과 정확히 일치합니다. 따라서 대시보드에서 관찰된 '최대 동시 처리 16개'는 시스템의 의도된 동작임이 검증되었습니다.