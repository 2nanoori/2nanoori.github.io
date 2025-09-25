---
title: "Scaling Time Series Data Processing: Discovering TSDB Limits in Practice (Part 3)"
excerpt: "The journey from theory to reality: implementing the M×N strategy, discovering the real TSDB bottlenecks, and overcoming system-level limitations through infrastructure scaling."
categories:
  - System Architecture
  - Work Experience
tags:
  - Time Series
  - Data Processing
  - System Design
  - Performance Optimization
  - TSDB
  - Load Testing
  - Virtual Threads
  - Memory Optimization
  - Scalability
  - Architecture Patterns
toc: true
toc_sticky: true
date: 2025-09-25
last_modified_at: 2025-09-25
---

{% include mermaid.html %}

## Prologue: Testing TSDB Limits in the Real World

In [Part 1]({{ site.baseurl }}/scaling-timeseries-data-processing-mx-n-strategy-part1), we introduced the M×N scaling strategy designed to test TSDB performance limits. [Part 2]({{ site.baseurl }}/scaling-timeseries-data-processing-mx-n-strategy-part2) chronicled how team feedback led us to choose a pragmatic direct writer approach for our PoC.

Now comes the moment of truth: **actually testing our TSDB's limits and discovering where the real bottlenecks lie**.

What we discovered fundamentally changed our understanding of system performance. The bottlenecks weren't where we expected them to be, and the solutions required thinking beyond application-level optimizations. This is the story of how we uncovered the real limitations of our time series database and learned to scale beyond them.

## Chapter 1: Preparing for TSDB Limit Testing

### The TSDB Testing Strategy

Our goal was clear: **push the TSDB to its limits and discover where it breaks**. Using the M×N strategy from Part 1, we planned to systematically increase data load until we found the system's true bottlenecks.

The theoretical expectations were straightforward:

- **Base throughput**: ~22M metrics/minute
- **M=6 (10-second intervals)**: 22M × 6 = 132M expected
- **M=12 (5-second intervals)**: 22M × 12 = 264M expected

### The First Obstacle: Application Bottlenecks

### The Sobering Test Results

| Test | Parallelism | M | Expected/min | Actual/min | Efficiency |
|------|-------------|---|-------------|------------|------------|
| 1 | 4 | 6 | 130.2M | ~73M | **56%** |
| 2 | 6 | 6 | 130.2M | ~104M | **80%** |
| 3 | 6 | 12 | 260.4M | ~130M | **50%** |
| 4 | 6 | 20 | 434M | ~130M | **30%** |

**The harsh reality**: As we increased the multiplication factor (M), efficiency plummeted dramatically. We hit a **processing ceiling at approximately 140-145M metrics per minute**, regardless of hardware improvements.

### Initial Bottleneck Analysis

The TSDB cluster monitoring revealed that our database wasn't the limiting factor:
- **CPU Utilization**: 20-40%
- **Peak Ingestion Rate**: 18M datapoints/min
- **Memory/Disk**: All within normal ranges

**The Realization**: Before we could test TSDB limits, we had to eliminate application-level bottlenecks that were preventing us from generating sufficient load.

**The Priority Shift**: Our focus temporarily moved from "testing TSDB limits" to "ensuring our data generator could actually stress the TSDB."

## Chapter 2: Eliminating Application Bottlenecks

### Problem #1: Synchronous I/O Blocking

**Diagnosis**: The biggest issue was that database write operations were blocking the Kafka message processing threads. Threads would send data to the database and wait (blocking) for responses, preventing them from fetching new data from Kafka during this time.

**Before: Synchronous Architecture**

```mermaid
<div class="mermaid">
graph TD
    subgraph "Consumer Thread"
        A[Kafka Poll] --> B[Data Amplification]
        B --> C[timeSeriesService.write()]
        C -- "I/O Wait (seconds)" --> D[Write Complete]
        D --> A
    end
</div>
```

**After: Asynchronous Architecture with Virtual Threads**

```mermaid
<div class="mermaid">
graph TD
    subgraph "Consumer Thread"
        A[Kafka Poll] --> B[Data Amplification]
        B --> Q[Submit Write Task]
        Q --> A
    end

    subgraph "Virtual Thread Pool"
        Q --> VT1[Virtual Thread 1: Write Batch 1]
        Q --> VT2[Virtual Thread 2: Write Batch 2]
        Q --> VTx[Virtual Thread N...]
    end
</div>
```

**Implementation**: We leveraged Java 21's Virtual Threads to make write operations asynchronous. The main thread delegates write tasks to a virtual thread executor and immediately returns to process the next batch of data.

<details>
<summary><b>Code Example: Asynchronous Write Implementation</b></summary>

```java
// Before: Synchronous blocking write
try {
    if(!timeSeries.isEmpty()) {
        timeSeriesService.write(storageTier, timeSeries); // Blocks here
    }
} catch (Exception e) {
    log.error("Write failed", e);
    // Retry logic...
}

// After: Asynchronous non-blocking write  
if (!timeSeries.isEmpty()) {
    writerExecutor.submit(() -> {
        int retries = writeRetryCount;
        while (retries >= 0) {
            try {
                timeSeriesService.write(storageTier, timeSeries);
                writtenCounter.addAndGet(timeSeries.size());
                break; // Success
            } catch (Exception e) {
                retries--;
                log.error("Write failed. Retries left: {}", retries, e);
                if (retries >= 0) {
                    Thread.sleep(1000); // Retry delay
                }
            }
        }
    });
}
```
</details>

### Problem #2: Memory Allocation Storm

**Diagnosis**: Even after async conversion, high M values caused excessive CPU usage. The root cause was massive `HashMap` object creation during data amplification, leading to severe GC (Garbage Collection) pressure.

**Solution**: Pre-create all required `dimensions` Map variants once, then reuse them in loops. This reduced HashMap allocations from `M×N` times to just `N` times.

<details>
<summary><b>Code Example: Memory Optimization</b></summary>

```java
// Before: Creating new HashMap in every loop iteration
for (int i = 0; i < timeMultiplier; i++) {
    for (int instanceOffset = 0; instanceOffset < instanceMultiplier; instanceOffset++) {
        // New HashMap created M×N times
        String newInstanceNo = generateNewInstanceNo(originalInstanceNo, instanceOffset);
        TimeSeries expanded = original.copyWithTimestampAndInstanceNo(newTimestamp, newInstanceNo);
        result.add(expanded);
    }
}

// After: Pre-create dimension variants (N times) and reuse
List<Map<String, String>> dimensionVariants = new ArrayList<>(instanceMultiplier);
for (int instanceOffset = 0; instanceOffset < instanceMultiplier; instanceOffset++) {
    Map<String, String> variant = new HashMap<>(baseDimensions);
    variant.put("instanceNo", generateNewInstanceNo(originalInstanceNo, instanceOffset));
    dimensionVariants.add(variant);
}

// Main loop reuses pre-created Maps
for (int i = 0; i < timeMultiplier; i++) {
    for (Map<String, String> dims : dimensionVariants) {
        TimeSeries expanded = TimeSeries.builder()
                .timestamp(newTimestamp)
                .dimensions(dims) // Reuse, don't recreate
                .value(original.getValue())
                .build();
        result.add(expanded);
    }
}
```
</details>

### Application Optimization Results

- **M=6 efficiency improved from 56% to ~90%**
- **Achieved stable ~120M metrics/minute processing**
- **Application-level bottlenecks eliminated**

**Mission Accomplished**: Our data generator was now capable of properly stressing the TSDB. Time to discover the database's real limits.

## Chapter 3: Discovering the Real TSDB Bottleneck

### The True Test: Pushing TSDB to Its Limits

With our application optimized, we could finally conduct the real test: **finding the TSDB's breaking point**. We pushed harder with M=12 to generate 260M+ metrics per minute.

**The Surprising Result**: Instead of improvement, performance **decreased** to around 100M/minute, and we started seeing `Slow write request` warnings lasting 7-9 seconds.

This was it—**we had found the TSDB's actual limit**. Now we needed to understand what was causing it.

### The Smoking Gun: Database Concurrency Limits

**TSDB Dashboard Evidence:**
1. **Ingestion component CPU usage**: Only ~30% - plenty of headroom
2. **Concurrent inserts**: Despite our service establishing ~200 connections and sending parallel requests, the database was **processing only 16 concurrent write operations maximum**

### Root Cause: Intentional System Protection

The bottleneck wasn't a bug—it was a **feature**. The TSDB's ingestion component deliberately limits concurrent processing to `CPU cores × 2` to protect itself from overload.

**Our Infrastructure:**
- **CPU cores**: 8
- **Calculated limit**: 8 × 2 = 16 concurrent operations
- **Observed behavior**: Maximum 16 concurrent inserts

**The Real Story**: Our hundreds of parallel requests were queuing up in the database's internal queue, waiting for one of the 16 available processing slots. This waiting time manifested as the 7-9 second "slow write request" delays we observed.

### The TSDB Limit Discovery: Mission Accomplished

**This was exactly what we set out to find**: the real limit of our TSDB system. Not a theoretical limit, but the actual, measurable constraint that would affect production workloads.

**Key Insights:**
- **TSDB Concurrency Limit**: 16 simultaneous write operations (CPU cores × 2)
- **Actual Bottleneck**: Database-level throttling, not application performance
- **Scalability Constraint**: Adding more application instances wouldn't help without scaling the database infrastructure

## Chapter 4: Scaling Beyond TSDB Limits

### Breaking Through the TSDB Limit

Having identified the TSDB's concurrency bottleneck, we now faced a new challenge: **how to scale beyond this fundamental limit**.

**The Challenge:**
- **TSDB Constraint**: vminsert concurrency limit (16 slots) was the hard ceiling
- **Infrastructure Reality**: Current VM servers (8 cores, 32GB) determined this limit
- **Scaling Question**: How do we increase TSDB throughput beyond this constraint?

### The Pragmatic Choice: Scale-Out with Better Hardware

Faced with these constraints, our team opted for a comprehensive infrastructure upgrade:

**Available Solution: High-Performance Physical Servers**
- **Discovery**: 8 idle PM servers (mhbase010~mhbase017) were available
- **Specifications**: 40 cores, 256GB RAM each (vs. current 8 cores, 32GB)
- **Strategic advantage**: Both compute and concurrency limits could be addressed

**Infrastructure Migration Plan:**
```yaml
# Before: VM Infrastructure
Current Servers: 8×VM (8 cores, 32GB)
vminsert concurrency: 8×2 = 16 slots per server
Single instance limit: ~120M metrics/min

# After: PM Infrastructure  
New Servers: 8×PM (40 cores, 256GB)
vminsert concurrency: 40×2 = 80 slots per server
Expected improvement: 5x theoretical capacity per server
```

### Why This Matters: Real-World Infrastructure Decisions

This decision illustrates several crucial principles in production system scaling:

1. **Resource Availability Over Theoretical Optimization**: Sometimes the best technical solution is to leverage available high-quality infrastructure rather than complex software optimizations.

2. **Holistic System Scaling**: We didn't just scale the data writer service—we scaled the entire stack (compute, memory, and database concurrency limits).

3. **Pragmatic Resource Utilization**: Repurposing idle high-specification servers provided immediate value without procurement delays.

**The Infrastructure-First Approach:**
- **Immediate impact**: 5x theoretical improvement in per-server capacity
- **Simplified scaling**: Higher-spec servers reduce the number of instances needed
- **Future-proofing**: Headroom for even more aggressive testing (M=12, M=20)

This approach validated our optimization work while providing the infrastructure foundation needed for the next phase of testing.

## Chapter 5: Lessons Learned and Takeaways

### Technical Insights

1. **Async I/O is Critical**: Virtual Threads provided an elegant solution for I/O-bound operations
2. **Memory Allocation Matters**: Object reuse patterns can dramatically reduce GC pressure
3. **System Limits are Real**: External system constraints often become the final bottleneck
4. **Monitoring is Key**: Dashboard analysis was crucial for identifying the true bottleneck

### Architectural Principles

1. **Optimize Layer by Layer**: Start with application, then move to system level
2. **Measure Everything**: Performance assumptions without data are dangerous
3. **Respect System Boundaries**: Understanding external system limits is crucial
4. **Scale Horizontally**: When hitting hard limits, scale the entire system

### The Pragmatic Approach Vindicated

Our Part 2 decision to choose the direct writer approach over complex stream processing proved wise:

- **Faster time to value**: We could focus on optimization rather than infrastructure setup
- **Easier debugging**: Bottlenecks were in familiar application code
- **Incremental improvement**: Each optimization showed measurable results
- **Production ready**: Built on existing, battle-tested components

## Conclusion: TSDB Limits Discovered and Conquered

The journey from the M×N strategy concept to successfully testing our TSDB's limits taught us invaluable lessons about where real-world bottlenecks hide and how to systematically uncover them.

**Mission Accomplished: TSDB Limits Identified**
- **Discovered the Real Bottleneck**: TSDB concurrency limits (16 simultaneous operations)
- **Measured Actual Capacity**: ~120M metrics/minute per server under optimal conditions
- **Identified Scaling Path**: Infrastructure upgrade increases capacity to 400M+ metrics/minute (5x improvement)
- **Validated Testing Strategy**: M×N approach successfully stressed the system to reveal true constraints

**The Real Value**: This wasn't just about achieving high throughput—it was about **understanding our system's fundamental limits**. We now know exactly where our TSDB will break under load and how to scale beyond that point.

**The TSDB Testing Success**: The M×N strategy succeeded in its primary mission: revealing the true performance characteristics of our time series database. We discovered that the bottleneck wasn't where we expected (application layer) but where it actually mattered (database concurrency limits).

**For Production Planning**: These insights directly translate to capacity planning. We now know that each 8-core TSDB node can handle ~120M metrics/minute, and scaling requires either more nodes or higher-spec hardware with increased concurrency limits.

---

**What's Next?** With our TSDB limits clearly understood and scaling strategies validated, we're ready to tackle the next challenge: optimizing for cardinality explosion and complex query patterns. But that's a story for another day.

---

*This concludes our three-part series on testing TSDB limits through scaled time series data processing. From initial strategy through team collaboration to discovering real-world database constraints, we've covered the complete journey from theoretical concept to practical system understanding.*
