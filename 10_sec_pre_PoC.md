Hi.  @dean-zhu @hj-ji .

Today, I’ve been testing a data generator for a 10-second PoC. Please check it and feel free to share your thoughts.

# Pre-Test
## Performance Pre-Test Conclusion

### 1. Optimal Configuration Identified
According to the comprehensive pre-test results, the **optimal configuration** is:
- **M=6** (10-second intervals)
- **N=1** (single instance multiplier)
- **time-offset = 0~30** (30-second time window)
- **Efficiency: 100%+** (75M actual vs 60.2M expected)

**Complete Test Matrix**
| Test | Parallelism | M | N |time-offset-start|time-offset-end| Expected | Actual | Efficiency | JVM Heap |
|------|-------------|---|---|----------|----------|----------|---------|------------|----------|
| 1 | 4 | 1 | 1 | 0 | 60 | 21.7M | 21.7M | **100%** | 8GB |
| 2 | 4 | 6 | 1 |0 | 60 | 130.2M | ~73M | 56% | 8GB |
| 3 | 6 | 6 | 1 |0 | 60 | 130.2M | ~104M | 80% | 8GB |
| 4 | 6 | 12 | 1 |0 | 60 | 260.4M | ~130M | 50% | 8GB |
| 5 | 6 | 20 | 1 |0 | 60 | 434M | ~130M | 30% | 8GB |
| 6 | 6 | 20 | 1 |0 | 60 | 434M | ~142M | 33%| 10GB |
| 7 | 8 | 20 | 1 |0 | 60 | 434M | ~142M | 33% |  10GB |
| 8 | 6 | 60 | 1 |0 | 60 | 1300M | ~107M | 8% |  10GB |
| 9 | 6 | 20 | 1 | 0 | 30 | 217M | ~125M | 57% | 10GB |
| 10 | **6** | **6** | **1** | **0** | **30** | **60.2M** | **75M** | **100%** | **10GB** |
| 11 | 6 | 12 | 1 | 0 | 30 | 145M | ~105M | 72% | 10GB |

### 2. Performance Bottleneck Analysis
We identified a **clear throughput ceiling at ~140-145M metrics/minute** across multiple test scenarios:
- M=12: ~130M (50% efficiency)
- M=20: ~142M (33% efficiency)
- M=60: ~107M (8% efficiency)

This ceiling persists regardless of:
- JVM heap size increases (8GB → 10GB)
- Parallelism scaling (6 → 8)
- Time multiplier increases (M=12 → M=60)

**2.1. VictoriaMetrics Cluster Status**
- Monitoring Evidence
    - Resource Utilization: 20 ~ 40 %
    - Peak Ingestion Rate: 18M datapoints/min
    - Active Time Series 600M
    - Memory/CPU/Disk: All within normal operation ranges
- Conclusion: Victoria Metrics is not the limiting factor

![image](https://media.oss.navercorp.com/user/16081/files/e2d2b2e5-3485-4819-8c28-994c473726aa)
![image](https://media.oss.navercorp.com/user/16081/files/12821e71-02a6-41e1-8406-6e43d537e4cb)

**2.2. Application-Level Bottlenecks Identified**
- Data Transmission Bottleneck(M=12~20)
    - Symptoms: Plateau at ~130~140M metrics/min
    - Root Cause: Network serialization/batching inefficiency
    - Evidence: VM cluster shows capacity for higher ingestion
- CPU Processing Bottleneck (M=60)
    - Symptoms: 92% CPU usage, performance drops to 107M
    - Root Cause: Data generation overhead (60x multiplication)
    - Evidence: JVM monitoring shows a CPU spike during M=60 tests
- Memory Bottleneck (M=20+ with 8GB)
    - symptoms: PoC Transform initialization failure
    - Root Cause: Insufficient heap for 20x data multiplication
    - soulution: Resolved with 10GB heap
      ![image](https://media.oss.navercorp.com/user/16081/files/a89c8361-6872-40a5-b595-c51935373c62)
      ![image](https://media.oss.navercorp.com/user/16081/files/a81b47c3-cf1f-450c-8379-e0d0c3f8c5e1)

### 3. Recommendations
We need more servers for the 'Optimal Configuration'

```yaml
# Optimal configuration for current infrastructure
poc:
  scaling:
    time-multiplier: 6        # 10-second intervals
    time-offset-start: 0
    time-offset-end: 30       # 30-second window
    instance-multiplier: 1

# JVM settings
-Xms10g -Xmx10g
```


___

### Test
> M: Time interval multiplier
60 seconds → 10 seconds: M = 6 (6x increase)
60 seconds → 5 seconds: M = 12 (12x increase)
60 seconds → 1 second: M = 60 (60x increase)
N: Data volume multiplier (data volume expansion through instanceNo * N from existing tests)

1. parallelism 4, 1m interval (default) M = 1, N = 1
```
[2025-09-18 10:15:07.292][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesLabelInNameServiceImpl - [Write TimeSeries] statistics => 21777247 (21.777248M) metrics are written
[2025-09-18 10:16:07.292][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesLabelInNameServiceImpl - [Write TimeSeries] statistics => 21690263 (21.690264M) metrics are written
[2025-09-18 10:17:07.292][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesLabelInNameServiceImpl - [Write TimeSeries] statistics => 21861923 (21.861923M) metrics are written
[2025-09-18 10:18:07.292][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesLabelInNameServiceImpl - [Write TimeSeries] statistics => 21776556 (21.776556M) metrics are written
```
2. parallelism 4, 10sec interval M = 6, N = 1
```
[2025-09-18 10:25:56.730][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 64604058 (64.60406M) metrics are written
[2025-09-18 10:26:56.730][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 70286208 (70.28621M) metrics are written
[2025-09-18 10:27:56.730][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 78351876 (78.351875M) metrics are written
[2025-09-18 10:28:56.730][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 78128754 (78.12875M) metrics are written
```

3. paralleism 6, 10sec interval M = 6, N = 1
```
[2025-09-18 11:04:52.111][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 105576882 (105.57688M) metrics are written
[2025-09-18 11:05:52.111][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 101174928 (101.17493M) metrics are written
[2025-09-18 11:06:52.111][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 104677590 (104.67759M) metrics are written
[2025-09-18 11:07:52.111][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 106515066 (106.51506M) metrics are written
```

4. paralleism 6, 5sec interval M = 12, N = 1
```
[2025-09-18 11:42:42.416][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 130817124 (130.81712M) metrics are written
[2025-09-18 11:43:42.442][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 131156724 (131.15672M) metrics are written
[2025-09-18 11:44:42.416][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 129157404 (129.15741M) metrics are written
[2025-09-18 11:45:42.416][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 128725092 (128.72508M) metrics are written
```

5. parallelism 6, 3sec interval M= 20, N = 1
```
[2025-09-18 14:07:04.355][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 130498940 (130.49895M) metrics are written
[2025-09-18 14:08:04.355][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 131433880 (131.43388M) metrics are written
[2025-09-18 14:09:04.355][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 129433040 (129.43304M) metrics are written
[2025-09-18 14:10:04.355][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 128031880 (128.03188M) metrics are written
``` 

6. parallelism 6, 3sec interval M= 20, N = 1
   Increase JVM heap size 8 GB -> 10 GB
```
[2025-09-18 14:37:41.323][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 143635960 (143.63597M) metrics are written
[2025-09-18 14:38:41.323][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 144427280 (144.42728M) metrics are written
[2025-09-18 14:39:41.323][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 140833800 (140.83379M) metrics are written
[2025-09-18 14:40:41.323][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 142270800 (142.2708M) metrics are written
```
7. parallelism 8, 1sec interval M= 60, N = 1
   Increase JVM heap size 8 GB -> 10 GB
```
[2025-09-18 15:11:15.906][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 145507000 (145.507M) metrics are written
[2025-09-18 15:12:15.906][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 146565240 (146.56525M) metrics are written
[2025-09-18 15:13:15.906][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 145709880 (145.70988M) metrics are written
[2025-09-18 15:14:15.906][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 146930660 (146.93065M) metrics are written
```
8. parallelism 6, 1sec interval M= 60, N = 1
```
[2025-09-18 15:37:15.650][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 108824040 (108.82404M) metrics are written
[2025-09-18 15:38:15.558][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 107607480 (107.60748M) metrics are written
[2025-09-18 15:39:15.558][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 103861260 (103.86127M) metrics are written
[2025-09-18 15:40:15.607][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 107003760 (107.00376M) metrics are written
```

9. parallelism 6, 3sec interval M= 20, N = 1, time-offset = 0 ~ 30
```
[2025-09-18 16:16:47.324][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 123761340 (123.761345M) metrics are written
[2025-09-18 16:17:47.324][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 126569980 (126.569984M) metrics are written
[2025-09-18 16:18:47.324][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 126327420 (126.32742M) metrics are written
[2025-09-18 16:19:47.324][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 125038940 (125.03894M) metrics are written
```

10. parallelism 6, 10sec interval M= 6, N = 1, time-offset = 0 ~ 30
```
[2025-09-18 16:29:17.095][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 75889878 (75.88988M) metrics are written
[2025-09-18 16:30:17.095][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 74675025 (74.675026M) metrics are written
[2025-09-18 16:31:17.095][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 66302553 (66.30255M) metrics are written
[2025-09-18 16:32:17.095][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 71904804 (71.9048M) metrics are written
```

11. parallelism 6, 5sec interval M= 12, N = 1, time-offset = 0 ~ 30
```
[2025-09-18 16:45:02.653][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 105961374 (105.96137M) metrics are written
[2025-09-18 16:46:02.653][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 99988068 (99.98807M) metrics are written
[2025-09-18 16:47:02.653][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 103340190 (103.340195M) metrics are written
[2025-09-18 16:48:02.653][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 105118602 (105.1186M) metrics are written
```
- tsdb-writer
  ![image](https://media.oss.navercorp.com/user/16081/files/bcfb6a0c-2689-4257-a1a3-280558fbfece)

- [vm](http://10.179.28.113:3000/d/oS7Bi_0Wz/victoriametrics-cluster?from=now-3h&to=now&var-ds=bea4zefa6y5tsf&var-job=$__all&var-job_insert=$__all&var-job_select=$__all&var-job_storage=$__all&var-instance=$__all&var-adhoc=)
  ![image](https://media.oss.navercorp.com/user/16081/files/e324af1e-e447-4edc-b9ce-e50cb9f80438)






# Next Test
- 테스트 해보니 Heap Size를 높이면 될까싶어서 다시 한번 테스트를 실행
- 실행 옵션 
```
13:33:41.303 >>>>> Execute command [/home1/irteam/scripts/tsdb-writer/springboot.sh start] with user [irteam]
13:34:11.872 >>>>> Execute command success
>>>>> pid [99911], exit code [0]
>>>>> stdout:
springboot.sh version: 1.4.1
Read configurations from /home1/irteam/scripts/tsdb-writer/springboot.props
------------------------------------------
LOG_FOLDER="/home1/irteam/logs/tsdb-writer"
JAVA_OPTS="-server -Xms16g -Xmx16g -Xlog:gc*:file=$LOG_FOLDER/gc.log:time,uptime,level,tags:filecount=5,filesize=10M -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=$LOG_FOLDER/heap-was1.hprof"
EXEC_FILE="/home1/irteam/deploy/tsdb-writer/doc_base/libs/tsdb-writer-0.0.1-SNAPSHOT.jar"
RUN_ARGS="--spring.profiles.active=nbp-stage,hot \
--kafka.parallel-consumer.parallelism=6 \
--poc.scaling.enabled=true \
--poc.scaling.time-multiplier=6 \
--poc.scaling.instance-multiplier=1 \
--poc.scaling.time-offset-start=0 \
--poc.scaling.time-offset-end=60"
CONSOLE_LOG="$LOG_FOLDER/springboot.log"
KILL_TRIES=5
KILL_INTERVAL_SECONDS=5
TEST_URL="http://localhost:8080/monitor/l7check"
WAIT_SECONDS_BEFORE_TEST=20
TEST_TRIES=7
TEST_INTERVAL_SECONDS=5------------------------------------------
Starting Spring Boot process on 8080 port.
Waiting for the Spring Boot process to start.(20 seconds)
Checking HTTP port. (1/7)
The Spring Boot process has started successfully.

```

- 결과
```

[2025-09-23 13:34:45.967][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 101192556 (101.19256M) metrics are written
[2025-09-23 13:35:45.967][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 100397604 (100.3976M) metrics are written
[2025-09-23 13:36:45.967][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 105010350 (105.01035M) metrics are written
[2025-09-23 13:37:45.966][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 108819384 (108.81938M) metrics are written
[2025-09-23 13:38:45.966][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 108240768 (108.24077M) metrics are written
[2025-09-23 13:39:45.968][INFO ][scheduling-1] c.n.n.c.s.i.i.TimeSeriesStandardServiceImpl - [Write TimeSeries] statistics => 108762312 (108.762314M) metrics are written
```