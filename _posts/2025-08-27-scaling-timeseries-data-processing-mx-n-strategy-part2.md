---
title: "Scaling Time Series Data Processing: A Pivot to Pragmatism (Part 2)"
excerpt: "The story of how team feedback transformed a complex stream processing architecture into a simple, pragmatic solution for our PoC, with a deep dive into the trade-offs and lessons learned."
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
  - Architecture Patterns
  - Collaboration
toc: true
toc_sticky: true
date: 2025-08-27
last_modified_at: 2025-08-27
---

{% include mermaid.html %}

## Introduction

In [Part 1 of this series]({% post_url 2025-08-25-scaling-timeseries-data-processing-mx-n-strategy-part1 %}), I detailed a stream-native "M×N" strategy for scaling time series data. The core idea was an "Immediate Transformation" approach, designed to be implemented in a dedicated stream processing layer using Flink or Kafka Streams. It was a clean, theoretically sound architecture that separated the concerns of data transformation from data ingestion.

With the design complete, I presented it to my team for a reality check before building the Proof of Concept (PoC). This review process proved to be one of the most valuable stages of the project. The feedback I received led to a significant—and ultimately, much better—pivot in our implementation strategy.

This post tells the story of that pivot. It's a deep dive into how collaborative refinement, a healthy debate on trade-offs, and a focus on pragmatism led us away from a complex new system and toward a simpler, more effective solution that was right under our noses.

## The Reality Check: Idealism vs. Operational Reality

My initial proposal was met with thoughtful questions that quickly moved from the "what" to the "how." My colleagues, with their deep operational experience, immediately began stress-testing the idea against the friction of our real-world environment.

My colleague **Colleague A**, who has several more years of experience with this service than I do, raised two critical points:

1.  **Deployment Overhead**: "A new Flink job for a PoC requires cluster provisioning, configuration management, and ongoing maintenance. This adds significant complexity and time before we can even start testing. Is this the right approach for a quick validation?"
2.  **Network Latency Problems**: "Our tsdb-writer uses a custom parallel consumer because standard consumers can't handle the network delays between our data centers. If Flink runs in a different IDC, it would face the same issue."

This second point was crucial. It highlighted a practical constraint that I hadn't fully considered: the existing infrastructure was already optimized for our specific network environment.

Building on this, **Colleague B**, who also has extensive experience with this service, questioned the resource implications and suggested a simpler alternative.

## The Alternative: Scaling the `data writer service` Directly

**Colleague B**'s idea was elegant in its simplicity: "Instead of adding a new layer, why not embed the transformation logic into the existing `data writer service` and scale *that*?"

The proposal was to have each instance of the `data writer service` consume the same raw data, but be configured to generate a specific subset of the transformed data.

#### Implementation Sketch

For example, one instance could be configured to handle the first half of the new timestamps within a minute, and a second instance could handle the other half. The core logic inside the service would be modified to read its assigned timestamp offsets from its configuration and loop through them, generating a new metric for each one.

```java
// Inside the Data Writer Service's consumer logic
List<Integer> assignedOffsets = config.getAssignedTimestampOffsets();

public void process(Metric metric) {
    LocalDateTime baseTime = metric.getTimestamp().truncatedTo(ChronoUnit.MINUTES);
    
    for (int offset : assignedOffsets) {
        Metric transformed = metric.copy();
        transformed.setTimestamp(baseTime.plusSeconds(offset));
        
        // The N-dimension logic (cardinality increase) could also be applied here.
        
        emitToWriterQueue(transformed);
    }
}
```

This approach was not only simpler but also leveraged the existing, well-tested data writer service that was already handling our production workload.

## Head-to-Head: A Deeper Architectural Comparison

The discussion naturally led to a more formal comparison of the two approaches, weighing their trade-offs not just for the PoC, but for the long term.

| Aspect | Flink/Stream Processor Approach | Direct Writer Scaling Approach | Analysis |
|---|---|---|---|
| **PoC Implementation Speed** | Slow | **Fast** | No new infrastructure or deployment pipelines needed. |
| **Operational Risk** | High | **Low** | Introduces a new, unproven component vs. scaling a known, stable one. |
| **Resource Efficiency** | Low | **High** | Avoids duplicating data in a second Kafka topic, saving storage and network bandwidth. |
| **Architectural Purity** | **High** | Low | Concerns are cleanly separated vs. data writer handling both transformation and writing. |
| **Long-term Maintainability** | **Potentially Easier** | Potentially Harder | A dedicated component is easier to reason about, while the writer's logic might grow complex. |
| **Flexibility** | High | **Medium** | Flink offers a richer ecosystem for complex transformations beyond this PoC's scope. |

For the PoC, the choice was obvious. The Direct Writer Scaling approach was faster, safer, and more resource-efficient. We acknowledged the trade-off: we were sacrificing a degree of architectural "purity" for a massive gain in pragmatism. For the limited scope of the PoC, this was the right call.

#### Visual Comparison of Approaches

<div class="mermaid">
graph TB
    subgraph "Flink Approach (Complex)"
        A1[Source Topic] --> B1[Flink Cluster 1]
        A1 --> B2[Flink Cluster 2]
        A1 --> B3[Flink Cluster N]
        B1 --> C1[Target Topic]
        B2 --> C1
        B3 --> C1
        C1 --> D1[TSDB Writer]
        D1 --> E1[TSDB]
        
        style A1 fill:#ffcccc
        style C1 fill:#ffcccc
        style B1 fill:#ffcc99
        style B2 fill:#ffcc99
        style B3 fill:#ffcc99
    end
    
    subgraph "Direct Writer Scaling (Simple)"
        A2[Source Topic] --> D2[Writer Instance 1<br/>00s, 10s, 20s]
        A2 --> D3[Writer Instance 2<br/>30s, 40s, 50s]
        D2 --> E2[TSDB]
        D3 --> E2
        
        style A2 fill:#ccffcc
        style D2 fill:#ccffcc
        style D3 fill:#ccffcc
        style E2 fill:#ccffcc
    end
    
    subgraph "Key Differences"
        F1["Flink: 2 Topics, N+1 Consumers<br/>Complex, Resource-Heavy"]
        F2["Direct: 1 Topic, 2 Consumers<br/>Simple, Resource-Efficient"]
        
        style F1 fill:#ffcccc
        style F2 fill:#ccffcc
    end
</div>

## Expanded Conclusion: Deeper Lessons from a Simple Pivot

This experience was more than just a technical decision; it was a valuable lesson in how team collaboration and practical constraints shape engineering decisions. The initial feeling of "my design was rejected" quickly gave way to a deeper appreciation for the collaborative process.

#### 1. The Goal is Not a Perfect Design, but a Successful Outcome
My initial design was clean, scalable, and followed best practices. It was, in a vacuum, a "good" design. But the goal of the PoC wasn't to build a perfect system; it was to **quickly and safely validate a hypothesis about our TSDB's limits.** The team's feedback didn't invalidate my design; it re-centered the discussion on the *actual goal*. The simpler approach was "better" because it served the project's immediate goal more effectively.

#### 2. Experience is the Ultimate Design Pattern
The most critical piece of information in the room—the network latency issues between our data centers (IDCs)—wasn't on a wiki or in a design doc. It was in the heads of the engineers who had lived through it. This "tacit knowledge" (knowledge that exists in people's heads but isn't written down) is an organization's most valuable, and often most under-utilized, asset. A design process that doesn't actively seek to surface this knowledge is doomed to repeat past mistakes. My proposal, by being concrete, acted as a catalyst that unlocked this crucial experience from my colleagues.

#### 3. Embrace the Role of "Facilitator of the Best Idea"
As engineers, it's easy to become attached to our own ideas. But true seniority lies not in having the best idea yourself, but in ensuring the best idea wins, regardless of its origin. By presenting a well-researched initial proposal, I created a high-quality starting point. By listening to the feedback and championing the simpler alternative, I transitioned from "owner of an idea" to "facilitator of the team's best solution." This shift in mindset is a critical step in an engineer's growth.

This journey from a complex ideal to a simple, pragmatic solution was a powerful reminder that the best architectures emerge through collaborative discussion and practical constraints. They are a blend of theoretical rigor and operational reality, and the process of finding that blend is what makes engineering a deeply human and rewarding discipline.

---
*All technical content in this article is based on actual production experience. Specific system names and configuration values have been generalized for security.*
