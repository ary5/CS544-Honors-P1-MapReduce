# Design Document — Honors Project 1: MapReduce
**Team Members:** Ami B., Ary S. and Ryan T.  
**Section:** Tyler Harter (Section 1)  
**Language:** Python  
**Special Feature:** Adaptive and Key-Aware Combiner Functions for Optimized Data Flow  

---

## 1. System Overview

Our project implements a Python-based MapReduce system that runs on a single VM using Docker Compose. It consists of a **client**, a **boss** node, and **four worker containers**, each limited to one CPU core.

We extend traditional MapReduce with **adaptive and key-aware combiner functions**, allowing workers to intelligently reduce intermediate data volumes while minimizing overhead. Our design aims to balance performance optimization, fault tolerance, and clarity of architecture.

---

## 2. Architecture

### Components
- **Client:**  
  CLI for uploading data, submitting jobs, and monitoring progress.
  
- **Boss:**  
  Coordinates all jobs. Splits input data, assigns map/reduce tasks, and monitors worker status.

- **Workers:**  
  Lightweight FastAPI services that execute map, reduce, and combiner functions. Workers read/write from shared storage and communicate task results via HTTP.

- **Shared Storage:**  
  Implemented using a **Docker volume** mounted across all containers **(Shared volume mount)** for simplicity and performance. Avoids HDFS overhead while maintaining shared accessibility.

---

## 3. Communication Model

All communication uses **HTTP REST APIs** via **FastAPI**.

- **Client → Boss:**
  - `/submit_job`: Submit a MapReduce job
  - `/status`: Query job status and results

- **Boss → Worker:**
  - `/map_task`: Assign a map task
  - `/reduce_task`: Assign a reduce task

All data is transmitted as JSON for transparency and easy debugging.

---

## 4. Data Flow

1. **Upload Data**
   ```bash
   python client.py upload input.txt /data/input
   ```

2. **Submit Job**
   ```bash
   python client.py submit        --mapper wordcount_mapper.py        --reducer wordcount_reducer.py        --input /data/input        --output /data/output        --num-maps 4        --num-reduces 2
   ```

3. **Execution Pipeline**
   - Boss splits input into chunks and dispatches them to workers.
   - Workers execute the `mapper()` on each chunk.
   - **Adaptive combiner** runs automatically when output exceeds a configurable threshold (e.g., 10k records).
   - **Key-aware combiner** pre-aggregates frequent (“hot”) keys to reduce shuffle size further.
   - Boss merges intermediate results, assigns reducers, and collects final output to `/data/output/`.

---

## 5. Data Formats

| Type | Format | Description |
|------|---------|-------------|
| Input | Plain text | One record per line |
| Intermediate | JSON | `{ "key": <string>, "values": [v1, v2, ...] }` |
| Output | Plain text | Final key-value pairs per line |

---

## 6. Combiner Design (Special Feature)

### Motivation
Traditional combiners reduce data volume but always run after each map phase. However, for small datasets, this adds unnecessary CPU cost. To make our system smarter, we designed **adaptive**, **key-aware**, and **streaming** combiners.

### Design Features

#### **1. Adaptive Combiner**
The worker dynamically decides whether to apply the combiner based on intermediate data size:
```python
if len(map_output) > COMBINER_THRESHOLD:
    combined_output = combiner(map_output)
else:
    combined_output = map_output
```
This ensures combiners only run when beneficial, optimizing CPU vs. I/O tradeoffs.

#### **2. Key-Aware Combining**
Workers identify “hot keys” using frequency counting. The combiner aggregates only for frequently occurring keys:
```python
counts = Counter(k for k, _ in map_output)
hot_keys = {k for k, v in counts.items() if v > HOT_KEY_THRESHOLD}
```
This improves load balancing and reduces shuffle bottlenecks for skewed datasets.

#### **3. Streaming (Incremental) Combining**
Combining occurs in small batches as map output is produced, lowering memory usage and overlapping computation with aggregation.

#### **4. Multi-Level Combining**
A second combiner stage can run before reducers start to merge intermediate outputs globally, mimicking multi-tier aggregation used in real systems.

---

## 7. Fault Handling and Reliability

- Workers periodically send **heartbeats** to the boss.  
- The boss reassigns uncompleted tasks after a configurable timeout.  
- Intermediate and final outputs are written atomically to prevent corruption.  

This ensures consistent job completion even under simulated worker failure.

---

## 8. Testing Plan

We will include both **functional** and **performance-driven** tests to validate correctness and demonstrate novelty.

### **A. Functional Tests**
- **Basic correctness:** WordCount outputs match expected values.  
- **Empty input:** System handles zero-length files gracefully.  
- **Multiple job submissions:** Independent runs don’t interfere.  
- **Combiner correctness:** Output identical with and without combiners.  
- **Edge cases:** Single key input, unusual characters, and large keys.

### **B. Advanced Performance and Research Tests**
| Test | Description | Metric |
|------|--------------|---------|
| **Combiner Effectiveness** | Compare shuffle data size and job time with/without combiners | Reduction ratio, time |
| **Adaptive Threshold Sweep** | Run jobs with thresholds (1k, 10k, 100k) | Find optimal tradeoff |
| **Hot-Key Stress Test** | Test datasets with key skew (e.g., 90% same key) | Combiner benefit under skew |
| **Streaming Combiner Test** | Compare peak memory vs. batch combiner | Memory, latency |
| **Fault Recovery Test** | Simulate worker crash mid-task | Output consistency |
| **Integration Test** | Use real dataset (Project Gutenberg texts) | End-to-end validation |

### **C. Combiner-Specific Unit Tests**
- **Idempotence Test:** Re-running combiner on same data yields same result.  
- **Partial Combining Test:** Ensure hot-key aggregation does not alter cold keys.  
- **Adaptive Trigger Test:** Validate that combiner invocation aligns with configured thresholds.

### **D. Visualization and Metrics**
We will generate plots showing:
- Job runtime vs dataset size  
- Data shuffled vs dataset size  
- Memory usage over time  
- Speedup (%) with combiners enabled  

All visualizations will be included in our performance evaluation notebook.

---

## 9. Performance Evaluation

Metrics to measure:
- Job execution time
- Intermediate data volume
- Network I/O (from Docker stats)
- Worker CPU/memory utilization

Plots generated via `matplotlib` will include:
- **Job time vs dataset size**
- **Data shuffled vs threshold**
- **Speedup (%) vs input scale**

---

## 10. Example Job: Word Count

**Mapper (`wordcount_mapper.py`):**
```python
def mapper(line):
    for word in line.strip().split():
        yield word.lower(), 1
```

**Combiner (`wordcount_mapper.py`):**
```python
def combiner(pairs):
    counts = {}
    for key, value in pairs:
        counts[key] = counts.get(key, 0) + value
    for key, value in counts.items():
        yield key, value
```

**Reducer (`wordcount_reducer.py`):**
```python
def reducer(key, values):
    yield key, sum(values)
```

**Output:**
```
the 42
mapreduce 19
project 11
```

---

## 11. Expected Outcome

- The system runs fully in Docker Compose.  
- Combiner functions reduce shuffle volume and runtime.  
- Adaptive and key-aware behavior improves efficiency across datasets.  
- Performance plots clearly show quantifiable gains.  
- Codebase is easy to reproduce, test, and extend.

---

## 12. Next Steps

- [ ] Finalize adaptive combiner logic  
- [ ] Implement key-aware combining mechanism  
- [ ] Add REST API fault tolerance  
- [ ] Implement benchmark visualization scripts  
- [ ] Finalize unit + stress test suites  
- [ ] Prepare for demo with real dataset and comparative plots

---

**In short:** Our adaptive, key-aware combiner framework brings intelligence to the classic MapReduce model, turning an academic project into a practical and data-driven performance system.
