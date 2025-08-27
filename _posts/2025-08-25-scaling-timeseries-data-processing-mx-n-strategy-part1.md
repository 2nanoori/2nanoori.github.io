---
title: "Scaling Time Series Data Processing: The M×N Strategy and a Stream-Based Approach (Part 1)"
excerpt: "A deep dive into the architectural challenges and solutions for scaling time series data ingestion, based on real-world production experience."
categories:
  - System Architecture
  - Work Experience
tags:
  - Time Series
  - Data Processing
  - System Design
  - Stream Processing
  - TSDB
  - Load Testing
  - Cardinality
  - Scalability
  - Performance Optimization
  - OOM
  - Timeseries Database
  - Architecture Patterns
toc: true
toc_sticky: true
date: 2025-08-25
last_modified_at: 2025-08-25
---

{% include mermaid.html %}

## Introduction

When operating time series data processing systems, we often face requirements to scale data ingestion. Two scenarios are particularly common:

1.  **Granular Monitoring**: The need to increase data density by narrowing a 1-minute collection interval to 1 second.
2.  **System Performance Testing**: The need to measure the limits of a Time Series Database (TSDB) by stress-testing it with a massive volume of data.

Initially, I took a simple approach: why not just replicate the existing data by the required multiplier? However, this "simple" method led to unexpected problems, forcing me to re-evaluate the challenge from a completely different perspective.

This article shares the story of that trial-and-error process and the improved architecture that resulted from it.

## The Common First Approach: Simple Multiplication

### Initial Requirement and Intuitive Solution

```
Current: Low-density metric data (e.g., N-minute intervals)
Target: High-density metric data (e.g., M-second intervals)
```

Faced with this, the most common first thought is to **"replicate existing data by the required multiplier."**

```java
// Initial implementation: Simple multiplication
public void processData(List<Metric> metrics) {
    List<Metric> expandedMetrics = new ArrayList<>();
    
    for (Metric metric : metrics) {
        // Generate data by the target multiplier
        for (int multiplier = 0; multiplier < TARGET_MULTIPLIER; multiplier++) {
            Metric duplicated = metric.copy();
            duplicated.adjustTimestamp(multiplier);
            expandedMetrics.add(duplicated);
        }
    }
    
    // Send all data to the sink at once
    for (Metric expanded : expandedMetrics) {
        sink.write(expanded); // OOM occurs due to the massive data volume!
    }
}
```

### Unexpected Problems

This simple approach quickly reveals serious issues:

**1. OutOfMemoryError (OOM)**
```
Existing data volume: X
Multiplier: N
Result: X * N items loaded into memory simultaneously → OOM
```

**2. Garbage Collection (GC) Pressure**
- Massive creation of temporary objects.
- Long GC pause times leading to system delays.

**3. Scalability Limitations**
- Exponential memory usage as the multiplier increases.
- Unusable for scenarios requiring even larger data volumes.

These problems lead to a fundamental question: **"Is this approach truly practical?"**

## Problem Analysis and a New Design Approach

### Limitations of the Existing Method

Analyzing the fundamental reasons for this approach's failure:

1.  **Memory-centric thinking**: Loading all data into memory before processing.
2.  **Batch processing perspective**: Treating streaming data as a batch.
3.  **Lack of realism**: An approach disconnected from a real-world operational environment.

**The key insight**: *"The problem wasn't the technical implementation, but the approach itself."*

### Root Cause Analysis: Two Different Test Objectives

Upon deeper analysis, the initial requirement actually concealed **two distinct test objectives**:

#### 1. The M-Dimension: Increasing Data Density
```
Objective: More granular monitoring.
Method: 1-minute intervals → 1-second intervals (temporal refinement).
Impact: More data points for the *same* time series.
Test Target: TSDB write throughput and storage capacity limits.
```

#### 2. The N-Dimension: Increasing Cardinality
```
Objective: System performance testing (measuring TSDB limits).
Method: Diversifying unique identifiers (e.g., server IDs, instance IDs).
Impact: An N-fold increase in the number of unique time series, i.e., N-fold cardinality.
Test Target: TSDB performance for indexing, metadata processing, and label-based queries.

Cardinality = The number of unique time series (a combination of metric name + labels).
※ The timestamp does not affect cardinality.
※ Different unique identifiers are treated as distinct time series.
```

The root cause of my initial failure was not understanding this distinction and focusing only on "increasing data volume."

This realization demanded a new approach—one that abandoned the batch-oriented mindset of "load everything into memory" and instead embraced **the essence of stream processing** while considering both M and N dimensions.

At this point, I decided to first focus on the more complex **M-dimension (increasing data density)** and evaluated two alternative approaches.

### New Approaches for the M-Dimension: Memory Holding vs. Immediate Transformation

> **Note**: The N-dimension (increasing cardinality) is relatively simple to implement. We can generate distinct time series by assigning unique identifiers in each processing cluster. Therefore, we will focus on the more complex M-dimension implementation here.

### Approach 1: The Memory Holding Method

The first idea was to achieve "perfect" time distribution.

```java
// Pseudocode: Memory Holding Method
public class WindowedProcessor {
    private Map<String, List<Metric>> oneMinuteBuffer;
    
    public void process(Metric metric) {
        // Collect data for a 1-minute window
        buffer.add(metric);
        
        if (windowComplete()) {
            // Dispatch at precise intervals (e.g., every 1 second)
            sendAt("14:00:00", createMetrics(buffer, 0));
            sendAt("14:00:01", createMetrics(buffer, 1));
            sendAt("14:00:02", createMetrics(buffer, 2));
            // ... (interval is configurable)
        }
    }
}
```

**Advantages:**
- Perfectly simulates the time distribution pattern of high-density monitoring.
- An accurate simulation.

**Disadvantages:**
- High memory usage: O(M × time_window).
- Requires complex timer and state management.
- Risk of memory explosion when scaling with the N-dimension.

### Approach 2: The Immediate Transformation Method

```java
// Pseudocode: Immediate Transformation Method
public class ImmediateProcessor {
    public void process(Metric metric) {
        LocalDateTime baseTime = metric.getTimestamp()
            .truncatedTo(ChronoUnit.MINUTES);
        
        // Immediately transform one metric into M metrics upon arrival
        for (int offset = 0; offset < INTERVAL_SECONDS; offset += TARGET_INTERVAL) {
            Metric transformed = metric.copy();
            transformed.setTimestamp(baseTime.plusSeconds(offset));
            emit(transformed);
        }
    }
}
```

**Advantages:**
- Memory efficient: O(M).
- Simple to implement.
- Excellent scalability.

**Disadvantages:**
- Creates a momentary burst of data rather than a distributed load over time.

## The Critical Insight: "The Essence of Streaming"

While deliberating between the two approaches, I had a key realization.

### Analyzing Real-World Data Reception Patterns

How does data actually arrive in a real-world monitoring environment?

```
Data flow in high-density monitoring:
Actual Arrival Time   Metric Data Content
──────────────────────────────────────────
12:00:03              server1_cpu (timestamp: 12:00:00)
12:00:07              server2_cpu (timestamp: 12:00:00)  
12:00:13              server1_cpu (timestamp: 12:00:10)
12:00:15              server3_cpu (timestamp: 12:00:00)
12:00:17              server2_cpu (timestamp: 12:00:10)
12:00:23              server1_cpu (timestamp: 12:00:20)
...

→ Characteristic: Each server sends data at its own, uncoordinated time.
→ Result: Data is received as a continuous and irregular stream.
```

**The Key Insight**: 
> "In a real streaming environment, data arrives individually and continuously. Is there any reason to buffer it and process it as a batch?"

This realization changed everything. In a real-world environment:
- Metrics arrive **individually**.
- They are naturally **distributed over time**.
- They are handled as a **stream**, not a bulk operation.

The **Immediate Transformation** method, despite its "bursty" nature, actually created a load pattern that was more similar to the real-world environment!

## Detailed Technical Review

### Memory Complexity Analysis

Let's accurately analyze the memory usage of the Immediate Transformation method.

```java
public void process(Metric input) {
    // M objects are created momentarily
    for (int i = 0; i < M; i++) {
        Metric transformed = input.copy(); // M objects
        emit(transformed);
    }
}
```

The conclusion is that it uses **O(M) memory**. Although M objects are created, they are emitted immediately and become eligible for garbage collection. This is far more efficient than the O(M × time_window) complexity of the holding method.

### Data Volume Impact Analysis

I was also asked, "Does increasing M also increase cardinality?"

**The correct answer**: Changing the timestamp interval **does not increase cardinality.**

```
Cardinality is determined solely by the combination of metric name + labels.
The timestamp is not included in the cardinality calculation.

Example:
- cpu_usage{server="web01"} → 1 time series
- Changing the collection interval from 1 minute to 10 seconds still results in only 1 time series.
```

However, the **number of data points certainly increases**, which impacts storage and processing performance.

## Final Architecture Decision

### Selected Method: Immediate Transformation

Ultimately, I chose the Immediate Transformation method.

<div class="mermaid">
flowchart LR
    A["Source Topic<br/>(Low-Density Metrics)"] --> B["Stream Processor<br/>(Transformation Engine)"]
    B --> |"Immediate M-fold Transformation<br/>(1 → M)"| C["Target Topic<br/>(High-Density Metrics)"]
    C --> D["TSDB<br/>(Time Series DB)"]
    
    subgraph "M-Dimension: Time Division"
        E["timestamp: T1<br/>metric{labels}: value"] 
        E --> F["timestamp: T1<br/>metric{labels}: value"]
        E --> G["timestamp: T1+Δ<br/>metric{labels}: value"]
        E --> H["timestamp: T1+2Δ<br/>metric{labels}: value"]
        E --> I["..."]
    end
    
    B -.-> E
    style A fill:#e1f5fe
    style B fill:#f3e5f5
    style C fill:#e8f5e8
    style D fill:#fff3e0
</div>

**N-Dimension Scaling**: Implemented by running processing engines in parallel.
- **Each processing engine**: Generates distinct time series with different label values.
- **Implementation**: Add a unique identifier to the labels.
- **Example**: `metric{server="web01"}` → `metric{server="web01", metric_id="1"}`, `metric{server="web01", metric_id="2"}`

→ **Result**: Generates N times the number of unique time series with different identifiers, even for the same base metric.

### Selection Rationale

1.  **Realism**: The pattern is identical to a real-world streaming environment.
2.  **Efficiency**: Minimizes memory usage.
3.  **Simplicity**: Avoids complex state management.
4.  **Scalability**: Stable even when scaling with the N-dimension.
5.  **Test Objective**: Achieves the goal of increasing the data volume effectively.

### The M×N Scaling Strategy

I designed a two-dimensional scaling strategy to control data volume and cardinality independently.

**Test Objective of Each Dimension:**
- **M-Dimension**: A load test for pure data throughput, measuring the TSDB's write performance and storage limits.
- **N-Dimension**: A load test for the TSDB's indexing and metadata handling, testing the limits of features that depend directly on cardinality.

This distinction is critical because a TSDB reacts differently to an increase in data points versus an increase in the number of unique series.

#### M-Dimension: Increasing Data Points

```java
// M-Implementation: Timestamp division
LocalDateTime baseTime = metric.getTimestamp().truncatedTo(ChronoUnit.MINUTES);

for (int offset = 0; offset < INTERVAL_SECONDS; offset += TARGET_INTERVAL) {
    Metric transformed = metric.copy();
    transformed.setTimestamp(baseTime.plusSeconds(offset));
    emit(transformed); // Generates M times the data points
}
```

**Effect**: 
- Cardinality Impact: **None** (same metric + labels).
- Data Volume Impact: **M-fold increase**.

#### N-Dimension: Increasing Cardinality

```java
// N-Implementation: Diversifying time series via unique identifiers
public class TimeSeriesIdentifierTransformer {
    private final int metricId;
    
    public TimeSeriesIdentifierTransformer(int metricId) {
        this.metricId = metricId;
    }
    
    public Metric transform(Metric input) {
        Metric transformed = input.copy();
        
        // Add a new unique identifier to the labels
        transformed.addLabel("metric_id", String.valueOf(metricId));
        
        return transformed; // Generates N-fold cardinality
    }
}
```

**Effect**:
- Cardinality Impact: **N-fold increase** (creates new time series).
- Data Volume Impact: **N-fold increase**.

#### The Integrated M×N Strategy

```java
// M×N Integrated Implementation
for (int timeOffset = 0; timeOffset < M; timeOffset++) {
    for (int identifierOffset = 0; identifierOffset < N; identifierOffset++) {
        Metric transformed = metric.copy();
        
        // M: Refine the timestamp (increase data density)
        transformed.setTimestamp(baseTime.plusSeconds(timeOffset * TARGET_INTERVAL));
        
        // N: Diversify the unique identifier (increase cardinality)
        transformed.addLabel("metric_id", 
            String.valueOf(identifierOffset));
        
        emit(transformed); // Total data volume is M×N
    }
}
```

**Final Effect**:
```
Total Data Volume = Original Volume × M × N
Total Cardinality = Original Cardinality × N (M has no effect)
```

### Specific Impact Analysis on the TSDB

Let's analyze how each dimension of scaling specifically affects the time series database.

#### Impact of M-Dimension Scaling on the TSDB

**Primary Load Points:**
- **Write I/O**: Inserting more data points into the same series.
- **Network Bandwidth**: M-fold increase in data transfer between the client and the TSDB.
- **Disk Storage**: M-fold increase in storage per time series.
- **Compression Efficiency**: Changes in compression ratios due to higher data density.

**Measurable Metrics:**
```
- Write throughput (points/sec)
- Disk usage growth rate
- Memory usage for write buffers
- Query response time for time-range queries
```

#### Impact of N-Dimension Scaling on the TSDB

**Primary Load Points:**
- **Index Memory**: Creating and maintaining indexes for new label combinations.
- **Metadata Management**: N-fold increase in overhead for discovering and managing series.
- **Label-based Search**: Performance of pattern matching, e.g., `{server="web01_virtual_*"}`.
- **Aggregate Queries**: N-fold increase in the number of series to process for `GROUP BY` operations.

**Measurable Metrics:**
```
- Memory usage for the series index
- Label query performance (milliseconds)
- Cardinality limit warnings
- Query planning time for complex aggregations
```

#### Integrated Load Testing Scenarios

The M×N strategy enables targeted testing scenarios:

```
Scenario 1: M=60, N=1  → High data density, existing cardinality
Scenario 2: M=1, N=100 → Existing density, high cardinality
Scenario 3: M=10, N=10 → Balanced load test
```

This allows us to identify **which part of the TSDB becomes a bottleneck first.**

## Lessons Learned

### 1. Understand the Essence of the Problem
It was crucial to see past the surface-level requirement of "increasing data volume" to understand the true goal: measuring the performance limits of the TSDB.

### 2. Strive for Realism
A design that reflects the characteristics of the actual environment can be more valuable than a theoretically perfect simulation.

### 3. The Importance of Trade-off Analysis
Finding the right balance between performance vs. complexity, perfection vs. practicality, and memory vs. accuracy was key.

### 4. Validation Through Collaboration
Discussions with colleagues helped uncover and correct technical details that I might have missed on my own.

### 5. The Value of an Incremental Approach
Starting with a simple approach and refining it as needed was more effective than trying to solve the entire complex problem at once.

## Conclusion

This experience taught me how critical it is to understand the **temporal nature of data** and the **essence of streaming** in the time series domain.

It's easy to fall into the trap of pursuing technical perfection at the expense of practicality, or vice versa. Through this process, I reaffirmed the importance of a development process that involves **identifying the core problem, considering the real-world environment, and validating with peers.**

---

**Part 2 Preview**

In Part 2, I will cover the process of implementing and validating this M×N strategy in a real Proof of Concept (PoC) environment:

- **Technology Selection**: A comparative analysis of Kafka Streams vs. Flink and the rationale for the final choice.
- **PoC Architecture Design**: The concrete implementation of the M×N strategy using the selected technology.
- **Performance Test Design**: Test scenarios and metrics for measuring the limits of our TSDB.
- **Validation**: The results of our hypothesis tested with real data, including its successes and limitations.

Was the "Immediate Transformation" approach able to prove our hypothesis in the PoC? Find out in Part 2.

---

*All technical content in this article is based on actual production experience. Specific system names and configuration values have been generalized for security.*