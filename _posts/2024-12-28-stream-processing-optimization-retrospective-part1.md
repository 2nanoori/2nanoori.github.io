---
title: "Do We Really Need Real-Time? Stream Processing System Optimization Retrospective (Part 1)"
date: 2024-12-28
categories:
  - System Architecture
  - Work Experience
tags:
  - Stream Processing
  - System Design
  - Architecture
  - Performance Optimization
  - Kafka
  - Data Engineering
  - Clean Architecture
  - Conway's Law
toc: true
toc_sticky: true
excerpt: "A deep dive into optimizing legacy stream processing systems by questioning the fundamental need for real-time processing and applying architectural principles."
---

{% include mermaid.html %}

## 1. Introduction

Recently, I had the opportunity to review a data processing system in production. Multiple real-time stream processing jobs were running, each serving different roles, but over time, the system had become increasingly complex and inefficient.

This led me to ask a fundamental question: **"Do we really need real-time processing?"** This question became the catalyst for reviewing our entire architecture, and I learned a great deal in the process. This post is a retrospective of the insights and improvement strategies gained during that journey.

## 2. Stream vs Batch Processing Design Principles

### 2.1 Theoretical Background of Data Processing Paradigms

In data processing system design, choosing between **stream processing vs batch processing** is a crucial architectural decision. As Martin Kleppmann presents in "Designing Data-Intensive Applications," each approach comes with different trade-offs.

**Stream Processing**
- **Advantages**: Low latency, real-time responsiveness, continuous processing
- **Disadvantages**: High complexity, difficult failure recovery, challenging accuracy guarantees
- **Applications**: Real-time notifications, fraud detection, real-time dashboards

**Batch Processing**
- **Advantages**: High throughput, accuracy guarantees, simple failure recovery
- **Disadvantages**: High latency, lack of real-time capabilities
- **Applications**: Daily reports, ETL jobs, large-scale analytics

### 2.2 Framework for Determining Real-Time Processing Necessity

Drawing from Google's "The Dataflow Model" paper and Apache Beam's design philosophy, I established the following criteria:

#### Event Time vs Processing Time Considerations
```
Event Time: When the data actually occurred
Processing Time: When the system processes the data
```
**When real-time processing is necessary:**
- **SLA requirements**: Responses within seconds to minutes are business-critical
- **Stateful computations**: Sliding windows, session-based aggregations, etc.
- **Event-driven triggers**: Immediate alerts when thresholds are exceeded
- **Interactive Analytics**: Real-time responses to user queries

**When batch processing is sufficient:**
- **Accuracy > Speed**: Financial reconciliation, regulatory reports, etc.
- **Large-scale processing**: TB-scale data batch transformations
- **Complex joins**: Complex relationships between multiple data sources
- **Periodic aggregations**: Daily/weekly/monthly reporting

### 2.3 Data Architecture Pattern Analysis

#### Lambda Architecture (Nathan Marz, 2011)

<div class="mermaid">
graph TB
    DS[Data Sources] --> SL[Speed Layer<br/>Real-time Stream]
    DS --> BL[Batch Layer<br/>Accurate Batch]
    
    SL --> |Approximate<br/>Low Latency| RT[Real-time Views]
    BL --> |Accurate Results<br/>High Latency| BV[Batch Views]
    
    RT --> SERVE[Serving Layer]
    BV --> SERVE
    SERVE --> |Unified Results| CLIENT[Client Queries]
    
    style SL fill:#fff3e0
    style BL fill:#e3f2fd
    style SERVE fill:#e8f5e8
</div>

**Lambda Architecture Characteristics:**
- **Dual pipelines**: Same logic implemented separately in stream and batch
- **CAP theorem trade-offs**: Pursuing both availability (real-time) and consistency (batch)
- **Cost of complexity**: Increased operational complexity to ensure accuracy and real-time capabilities

#### Kappa Architecture (Jay Kreps, 2014)

Referencing Hazelcast's [Kappa Architecture diagram](https://hazelcast.com/wp-content/uploads/2024/12/diagram-kappa-architecture.svg):

<div class="mermaid">
graph TB
    KAFKA[Kafka Cluster<br/>input_topic] --> SP1[Stream Processing<br/>job_version_n]
    KAFKA --> SP2[Stream Processing<br/>job_version_n+1]
    
    SP1 --> OUT1[output_table_n]
    SP2 --> OUT2[output_table_n+1]
    
    OUT1 --> DB[Serving DB]
    OUT2 --> DB
    
    DB --> APP[Application<br/>Queries]
    
    style SP1 fill:#fff3e0
    style SP2 fill:#e8f5e8
    style DB fill:#f3e5f5
</div>
**Kappa Architecture Characteristics:**
- **Single stream paradigm**: Unifying all processing into streams
- **Version management**: Reprocessing with new versions when stream logic changes
- **Simplicity limitations**: May be unsuitable for complex batch operations

#### Hybrid Approach in Practice

In reality, a **hybrid approach** is more practical than pure Lambda or Kappa:

<div class="mermaid">
graph TB
    DS[Data Sources] --> HOT[Hot Path<br/>Real-time Critical Data]
    DS --> WARM[Warm Path<br/>Near Real-time Analytics]
    DS --> COLD[Cold Path<br/>Batch Processing]
    
    HOT --> |Milliseconds| OLTP[(OLTP DB)]
    WARM --> |Minutes| STREAM[Stream Analytics]
    COLD --> |Hours/Days| OLAP[(OLAP DW)]
    
    style HOT fill:#ffebee
    style WARM fill:#fff3e0
    style COLD fill:#e3f2fd
</div>
### 2.4 Data Architecture Anti-patterns and Clean Architecture Principles

#### Clean Architecture and Data Processing Systems

Applying Robert C. Martin's Clean Architecture principles to data processing systems:

<div class="mermaid">
graph TB
    subgraph "External Layer"
        UI[UI/API]
        DB[(Database)]
        KAFKA[Message Queue]
    end
    
    subgraph "Interface Adapters"
        CTRL[Controllers]
        REPO[Repositories]
        PRESENT[Presenters]
    end
    
    subgraph "Application Business Rules"
        USECASE[Use Cases]
        STREAM[Stream Processors]
        BATCH[Batch Jobs]
    end
    
    subgraph "Enterprise Business Rules"
        ENTITY[Business Entities]
        RULE[Business Rules]
    end
    
    UI --> CTRL
    CTRL --> USECASE
    USECASE --> ENTITY
    REPO --> DB
    STREAM --> KAFKA
    
    style ENTITY fill:#e8f5e8
    style USECASE fill:#fff3e0
    style REPO fill:#f3e5f5
</div>
**Core Principles:**
- **Dependency Inversion**: Business logic doesn't depend on infrastructure
- **Separation of Concerns**: Real-time/batch processing methods are implementation details
- **Testability**: Core logic can be tested independently

#### Data Processing Anti-pattern Deep Analysis

**1. Over-Real-timing**
<div class="mermaid">
graph LR
    A[All Data] --> B[Real-time Stream]
    B --> C[Complex State Management]
    C --> D[High Operational Cost]
    
    style B fill:#ffebee
    style D fill:#ffebee
</div>- **Root cause**: Misconception that "real-time = better"
- **Martin Fowler's YAGNI principle violation**: You Aren't Gonna Need It
- **Solution**: Business value-based real-time necessity assessment

**2. Data Pipeline Spaghetti**
<div class="mermaid">
graph TB
    S1[Stream 1] --> T1[Topic A]
    S2[Stream 2] --> T1
    S3[Stream 3] --> T2[Topic B]
    T1 --> S4[Stream 4]
    T1 --> S5[Stream 5]
    T2 --> S4
    T2 --> S6[Stream 6]
    S4 --> T3[Topic C]
    S5 --> T3
    
    style T1 fill:#ffebee
    style T2 fill:#ffebee
    style T3 fill:#ffebee
</div>- **Problem**: Debugging difficulties due to complex dependencies
- **Clean Architecture violation**: High coupling, low cohesion
- **Solution**: Domain-based topic design, Event Sourcing patterns

**3. State Management Hell**
- **Problem**: Distributed state synchronization complexity
- **Example**: Multiple streams modifying shared state
- **Solution**: Apply CQRS (Command Query Responsibility Segregation)

**4. Technology-Driven Design**
- **Problem**: Technology stack prioritized over business logic
- **Example**: "Let's use streams because we have Kafka"
- **Solution**: Apply Domain-Driven Design (DDD)

#### Deeper Architectural Considerations

Your concerns go beyond Lambda/Kappa to more fundamental issues:

**1. Conway's Law and Organizational Structure**
- "System structure reflects organizational structure"
- Team boundaries become system boundaries, creating duplication

**2. Technical Debt and Evolutionary Architecture**
- Evolutionary Architecture (Neal Ford): Managing architectural change over time
- Incremental improvement vs big-bang refactoring

**3. Operational Complexity and Cognitive Load**
- Cognitive Load Theory (John Sweller): Limits of complexity developers can handle
- Higher system complexity increases error probability

**4. Cost Optimization and Performance Trade-offs**
- New constraints in the cloud era: Network costs, storage costs
- FinOps: Architectural decisions from a financial perspective

## 3. Current System Analysis

Analyzing the legacy system revealed the following architectural issues:

### 3.1 Duplicate Processing Structure Discovery

<div class="mermaid">
graph TB
    K[Message Queue] --> MP[Main Aggregation Processor]
    K --> AP[Auxiliary Data Processor]
    
    MP --> |Long-term Aggregation| DB[(Time Series DB)]
    AP --> |Short-term Raw Data| DB
    AP --> |Metadata| SEARCH[(Search Engine)]
    
    style MP fill:#e1f5fe
    style AP fill:#fff3e0
    style DB fill:#f3e5f5
    style SEARCH fill:#e8f5e8
</div>
**Issues:**
- **Main Aggregation Processor**: Aggregates data across multiple time units (seconds to days)
- **Auxiliary Data Processor**: Separately processes short-term raw data and metadata
- Duplicate consumption from the same message queue, wasting resources
- Operating separate streams just for short-term raw data processing

### 3.2 Streaming of Periodic Scan Jobs

<div class="mermaid">
graph TB
    TIMER[1-minute Cycle] --> SCAN[Event Scan Job]
    SCAN --> CACHE[(Cache Store)]
    SCAN --> CHECK{Condition Check}
    CHECK --> |On Match| NOTIFY[Send Notification]
    
    style SCAN fill:#fff3e0
    style CACHE fill:#f3e5f5
</div>
**Issues:**
- Function to send notifications when events reoccur based on user-configured conditions
- Batch-nature job that periodically scans cache store implemented as a stream
- Processing 1-minute delay-tolerant jobs with real-time streams

### 3.3 Resource Usage Tracking Duplicate Structure

<div class="mermaid">
graph TB
    STREAM[Real-time Usage Aggregation] --> |Short-term Units| USAGE[(Usage Statistics)]
    BATCH[Existing Batch Aggregation] --> |Long-term Units| REPORT[(Report Data)]
    
    style STREAM fill:#fff3e0
    style BATCH fill:#e3f2fd
</div>
**Issues:**
- Real-time aggregation feature for resource usage in short-term units
- Potential role overlap with existing batch aggregation system
- Increased complexity in usage data consistency management

## 4. Derived Improvement Solutions

### 4.1 Principle-Based Improvement Direction

Based on the analysis, I established the following improvement principles:

1. **Re-evaluate real-time necessity**: Review whether each function truly needs real-time processing
2. **Eliminate duplication**: Integrate components with identical roles
3. **Choose appropriate processing methods**: Apply clear criteria for real-time vs batch
4. **Reduce system complexity**: Remove unnecessary components to reduce operational burden

### 4.2 Specific Solutions

#### Solution 1: Convert Search Engine Synchronization to Batch
```
Current: Real-time stream synchronization of metadata to search engine
Improvement: Convert to daily batch for improved stability
Reason: Metadata queries prioritize accuracy over real-time capabilities
Effect: Reduced search engine load, improved synchronization stability
```
#### Solution 2: Convert Event Notifications to Batch
```
Current: Real-time stream cache scanning and notification sending
Improvement: Convert to 1-minute periodic batch job
Reason: 1-minute delay acceptable for notification function
Effect: Reduced stream processing complexity, resource savings
```
#### Solution 3: Integrate Short-term Data Processing
```
Current: Main system excludes specific time unit data processing
Improvement: Integrated processing of all time unit data in single system
Reason: Absorb auxiliary system's single role into main system
Effect: Reduced system count, improved data consistency
```
#### Solution 4: Integrate Usage Aggregation with Batch
```
Current: Separate real-time usage aggregation and batch reporting systems
Improvement: Integrate into existing batch aggregation system
Reason: Report data prioritizes accuracy over speed
Effect: Improved data consistency, eliminated duplicate logic
```
### 4.3 Improved Integrated Architecture

<div class="mermaid">
graph TB
    K[Message Queue] --> UP[Unified Processor]
    
    UP --> |All Time Unit Aggregation| DB[(Time Series DB)]
    
    BATCH[Batch Scheduler] --> META[Metadata Synchronization]
    META --> SEARCH[(Search Engine)]
    
    BATCH --> REMIND[Event Notification Processing]
    REMIND --> NOTIFY[Send Notifications]
    
    BATCH --> REPORT[Unified Reporting Processing]
    REPORT --> USAGE[(Usage Statistics)]
    
    style UP fill:#e8f5e8
    style BATCH fill:#e3f2fd
    style DB fill:#f3e5f5
    style SEARCH fill:#e8f5e8
</div>
### 4.4 Expected Effects

#### Cost Reduction
- **Infrastructure costs**: ~40% resource savings by eliminating 2 stream processing jobs
  - **Calculation basis**: 
    - Current: 5 stream processing instances (2 main + 3 auxiliary) × 2 CPU cores = 10 cores
    - Proposed: 3 unified instances × 2 CPU cores = 6 cores
    - Savings: (10-6)/10 = 40% CPU resource reduction
  - **Memory optimization**: Reduced JVM heap allocation from 20GB to 12GB total
  - **Network costs**: ~30% reduction in inter-service communication overhead
- **Operational costs**: Reduced operational burden through fewer monitoring targets
  - **Reduced monitoring complexity**: From 5 services to 3 services
  - **Simplified deployment pipeline**: 40% fewer deployment stages

#### Improved Stability
- **Data consistency**: Improved integrity by eliminating duplicate processing
- **Failure recovery**: Shortened incident response time through simplified system structure

#### Development Efficiency
- **Code complexity**: Improved maintainability by eliminating duplicate logic
- **Deployment complexity**: Reduced release risk by fewer deployment targets

#### Monitoring and Observability Improvements
- **Centralized metrics**: Unified dashboards for all processing components
- **Simplified alerting**: Reduced false positive alerts from duplicate systems
- **Distributed tracing**: Cleaner trace paths with fewer service hops
- **Log aggregation**: Consolidated log streams for easier debugging
- **SLA monitoring**: Simplified end-to-end latency tracking

## 5. Conclusion

Through this analysis, I realized the importance of asking **"Do we really need real-time?"** Just because something is technically possible doesn't mean everything needs to be processed in real-time. Finding the balance between business requirements and technical complexity is crucial.

I particularly learned the following lessons:

1. **Establish clear criteria**: Need clear criteria for choosing real-time vs batch
2. **Periodic architecture reviews**: Clean up technical debt accumulated over time
3. **Business value focus**: Focus on actual value rather than technical possibilities

### 5.1 Practical Checklist for Stream Processing System Evaluation

For teams considering similar optimizations, here's a practical checklist:

#### ✅ Real-time Necessity Assessment
- [ ] Can the business tolerate 1-minute delays?
- [ ] Are there strict SLA requirements under 10 seconds?
- [ ] Is this data used for immediate decision-making?
- [ ] Would batch processing compromise user experience?

#### ✅ System Complexity Review
- [ ] Are there duplicate processing pipelines?
- [ ] Can multiple streams be consolidated?
- [ ] Are failure recovery procedures overly complex?
- [ ] Is the monitoring overhead manageable?

#### ✅ Cost-Benefit Analysis
- [ ] What are the actual infrastructure costs per stream?
- [ ] How much operational overhead does each component add?
- [ ] Are the benefits proportional to the complexity?
- [ ] Can simpler alternatives achieve 80% of the value?

#### ✅ Data Quality Considerations
- [ ] Is data consistency more important than latency?
- [ ] Are there data freshness requirements by stakeholders?
- [ ] Can eventual consistency models work for your use case?
- [ ] How critical is exactly-once processing?

---

**Preview of Part 2**

In Part 2, I'll cover how these improvement strategies were discussed and evaluated, what difficulties were encountered during implementation, and what the final results were. I'll particularly share the consensus-building process, phased implementation strategy, and actual effectiveness measurement results.

---

*This article is based on real work experiences, with specific system names and configuration values generalized for security purposes.*

## References

- Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly Media.
- Martin, R. C. (2017). *Clean Architecture: A Craftsman's Guide to Software Structure and Design*. Prentice Hall.
- Fowler, M. (2002). *Patterns of Enterprise Application Architecture*. Addison-Wesley.
- Ford, N. et al. (2017). *Building Evolutionary Architectures*. O'Reilly Media.
- Akidau, T. et al. (2015). "The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost". VLDB.
- Kreps, J. (2014). "Questioning the Lambda Architecture". O'Reilly Radar.
- Marz, N. & Warren, J. (2015). *Big Data: Principles and best practices*. Manning Publications.
- Conway, M. E. (1968). "How Do Committees Invent?". Datamation.
- Hazelcast Kappa Architecture: https://hazelcast.com/wp-content/uploads/2024/12/diagram-kappa-architecture.svg
