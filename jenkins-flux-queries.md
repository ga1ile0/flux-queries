# Jenkins Flux Queries for Grafana

This document contains example Flux queries for visualizing Jenkins metrics stored in InfluxDB.

## 1. Executor Utilization Over Time

Displays the percentage of busy executors compared to total executors.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins")
  |> filter(fn: (r) => r._field == "busy_executors" or r._field == "total_executors")
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      r with
      utilization: float(v: r.busy_executors) / float(v: r.total_executors) * 100.0
    }))
  |> keep(columns: ["_time", "utilization"])
```

## 2. Current Busy vs Total Executors

Shows the current number of busy and total executors as a gauge or stat panel.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins")
  |> filter(fn: (r) => r._field == "busy_executors" or r._field == "total_executors")
  |> last()
```

## 3. Node Status Distribution

Displays the count of online vs offline Jenkins nodes.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "num_executors")
  |> group(columns: ["status"])
  |> last()
  |> group()
  |> count()
```

## 4. Node Memory Utilization

Shows memory usage percentage for each Jenkins node.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "memory_available" or r._field == "memory_total")
  |> pivot(rowKey: ["_time", "node_name"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      r with
      memory_used_percent: (float(v: r.memory_total) - float(v: r.memory_available)) / float(v: r.memory_total) * 100.0
    }))
  |> keep(columns: ["_time", "node_name", "memory_used_percent"])
```

## 5. Node Disk Space Available

Displays available disk space on each Jenkins node over time.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "disk_available")
  |> map(fn: (r) => ({ r with _value: r._value / 1073741824.0 }))
  |> keep(columns: ["_time", "node_name", "_value"])
```

## 6. Job Success Rate by Job Name

Calculates the success rate for each Jenkins job.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_job")
  |> filter(fn: (r) => r._field == "result_code")
  |> group(columns: ["name"])
  |> map(fn: (r) => ({
      r with
      is_success: if r._value == 0 then 1 else 0
    }))
  |> mean(column: "is_success")
  |> map(fn: (r) => ({ r with _value: r._value * 100.0 }))
```

## 7. Average Job Duration by Job Name

Shows the average duration of jobs grouped by job name.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_job")
  |> filter(fn: (r) => r._field == "duration")
  |> group(columns: ["name"])
  |> mean()
  |> map(fn: (r) => ({ r with _value: r._value / 1000.0 }))
```

## 8. Job Build Count by Result

Displays the total number of builds grouped by result status.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_job")
  |> filter(fn: (r) => r._field == "result_code")
  |> group(columns: ["result"])
  |> count()
```

## 9. Recent Failed Jobs

Lists the most recent failed job executions with their names and timestamps.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_job")
  |> filter(fn: (r) => r._field == "result_code")
  |> filter(fn: (r) => r._value == 1)
  |> group(columns: ["name"])
  |> last()
  |> keep(columns: ["_time", "name", "result"])
```

## 10. Node Response Time

Monitors the response time of each Jenkins node.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "response_time")
  |> group(columns: ["node_name"])
```

## 11. Total Builds Executed Over Time

Shows the cumulative count of all job executions.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_job")
  |> filter(fn: (r) => r._field == "number")
  |> group()
  |> count()
  |> cumulativeSum()
```

## 12. Swap Memory Usage by Node

Displays swap memory utilization percentage for each node.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "swap_available" or r._field == "swap_total")
  |> pivot(rowKey: ["_time", "node_name"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      r with
      swap_used_percent: (float(v: r.swap_total) - float(v: r.swap_available)) / float(v: r.swap_total) * 100.0
    }))
  |> keep(columns: ["_time", "node_name", "swap_used_percent"])
```

## 13. Jobs with Longest Duration

Identifies jobs with the longest execution times in the selected time range.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_job")
  |> filter(fn: (r) => r._field == "duration")
  |> group(columns: ["name"])
  |> max()
  |> group()
  |> top(n: 10)
  |> map(fn: (r) => ({ r with _value: r._value / 1000.0 }))
```

## 14. Executor Capacity by Node

Shows the number of executors available on each node.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "num_executors")
  |> group(columns: ["node_name"])
  |> last()
```

## 15. Job Failure Rate Over Time

Tracks the failure rate of all jobs over time as a time series.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_job")
  |> filter(fn: (r) => r._field == "result_code")
  |> aggregateWindow(every: 1h, fn: (tables=<-, column) =>
      tables
        |> map(fn: (r) => ({ r with is_failure: if r._value == 1 then 1 else 0 }))
        |> mean(column: "is_failure")
    )
  |> map(fn: (r) => ({ r with _value: r._value * 100.0 }))
```

---

## Jenkins Node Queries

### 16. Node Online/Offline Status Table
**Visualization Type:** Table

Displays the current status of all Jenkins nodes with their key metrics.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "num_executors")
  |> group(columns: ["node_name", "status", "arch"])
  |> last()
  |> keep(columns: ["_time", "node_name", "status", "arch", "_value"])
```

### 17. Node Disk Usage Percentage
**Visualization Type:** Time Series (Graph)

Shows disk usage trends for each node over time.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "disk_available")
  |> group(columns: ["node_name", "disk_path"])
  |> map(fn: (r) => ({
      r with
      _value: r._value / 1073741824.0
    }))
  |> keep(columns: ["_time", "node_name", "disk_path", "_value"])
```

### 18. Node Temp Space Available
**Visualization Type:** Bar Gauge

Monitors available temporary storage space on each node (in GB).

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "temp_available")
  |> group(columns: ["node_name", "temp_path"])
  |> last()
  |> map(fn: (r) => ({ r with _value: r._value / 1073741824.0 }))
  |> keep(columns: ["node_name", "temp_path", "_value"])
```

### 19. Node Response Time Comparison
**Visualization Type:** Bar Chart

Compares average response times across all nodes to identify slow performers.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "response_time")
  |> group(columns: ["node_name"])
  |> mean()
  |> group()
  |> sort(columns: ["_value"], desc: true)
```

### 20. Nodes with Low Memory Alert
**Visualization Type:** Table / Stat

Identifies nodes with less than 20% memory available (critical alert).

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "memory_available" or r._field == "memory_total")
  |> pivot(rowKey: ["_time", "node_name"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      r with
      memory_available_percent: float(v: r.memory_available) / float(v: r.memory_total) * 100.0
    }))
  |> filter(fn: (r) => r.memory_available_percent < 20.0)
  |> keep(columns: ["_time", "node_name", "memory_available_percent"])
```

### 21. Total Executor Capacity Across Nodes
**Visualization Type:** Stat / Gauge

Shows the total number of executors available across all online nodes.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "num_executors")
  |> filter(fn: (r) => r.status == "online")
  |> last()
  |> group()
  |> sum()
```

### 22. Node Architecture Distribution
**Visualization Type:** Pie Chart

Displays the count of nodes by architecture type (e.g., x86, ARM).

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "num_executors")
  |> group(columns: ["arch"])
  |> last()
  |> group()
  |> count()
```

### 23. Node Memory Available Over Time
**Visualization Type:** Time Series (Graph)

Tracks memory availability trends for all nodes (in GB).

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "memory_available")
  |> group(columns: ["node_name"])
  |> map(fn: (r) => ({ r with _value: r._value / 1073741824.0 }))
  |> keep(columns: ["_time", "node_name", "_value"])
```

### 24. Nodes by Source/Port
**Visualization Type:** Table

Groups and displays nodes by their Jenkins source and port.

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "num_executors")
  |> group(columns: ["source", "port", "node_name"])
  |> last()
  |> keep(columns: ["node_name", "source", "port", "status", "_value"])
```

### 25. Node Swap Memory Usage Over Time
**Visualization Type:** Time Series (Graph)

Monitors swap memory consumption for each node to detect memory pressure (in GB).

```flux
from(bucket: "metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_node")
  |> filter(fn: (r) => r._field == "swap_available" or r._field == "swap_total")
  |> pivot(rowKey: ["_time", "node_name"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      r with
      swap_used_gb: (float(v: r.swap_total) - float(v: r.swap_available)) / 1073741824.0
    }))
  |> keep(columns: ["_time", "node_name", "swap_used_gb"])
```

---

## Notes

- All queries use `v.timeRangeStart` and `v.timeRangeStop` for compatibility with Grafana's time range picker
- Duration values are converted from milliseconds to seconds where appropriate
- Disk space is converted to GB in query #5
- Result codes: 0=SUCCESS, 1=FAILURE, 2=NOT_BUILD, 3=UNSTABLE, 4=ABORTED
- Memory, disk, and swap values are converted to GB where appropriate for better readability
