# Grafana Flux Queries for Jenkins Metrics

This document contains 15 Flux queries for visualizing Jenkins metrics in Grafana.

---

## 1. Build Duration Over Time
**Visualization Type:** Time series (Line chart)

Shows the duration of completed builds over time for all jobs.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "duration")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> keep(columns: ["_time", "_value", "name"])
  |> yield(name: "mean")
```

---

## 2. Build Success Rate
**Visualization Type:** Gauge

Displays the percentage of successful builds in the selected time range.

```flux
success = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> filter(fn: (r) => r["_value"] == 0)
  |> count()
  |> group()
  |> sum()

total = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> count()
  |> group()
  |> sum()

union(tables: [success, total])
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({_value: float(v: r.result_code_success) / float(v: r.result_code_total) * 100.0}))
```

---

## 3. Recent Build Results Table
**Visualization Type:** Table

Shows a table of recent builds with job name, result, duration, and timestamp.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "duration" or r["_field"] == "result" or r["_field"] == "number")
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> keep(columns: ["_time", "name", "result", "duration", "number"])
  |> sort(columns: ["_time"], desc: true)
  |> limit(n: 20)
```

---

## 4. Total Builds Count
**Visualization Type:** Stat

Displays the total number of builds in the time range.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "number")
  |> count()
  |> group()
  |> sum()
```

---

## 5. Build Duration Heatmap
**Visualization Type:** Heatmap

Shows build durations as a heatmap, useful for identifying patterns.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "duration")
  |> aggregateWindow(every: 1h, fn: mean, createEmpty: false)
```

---

## 6. Failed Builds by Job
**Visualization Type:** Bar chart (Horizontal)

Shows count of failed builds per job.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> filter(fn: (r) => r["_value"] != 0)
  |> group(columns: ["name"])
  |> count()
  |> group()
  |> sort(columns: ["_value"], desc: true)
```

---

## 7. Node Memory Usage
**Visualization Type:** Time series (Area chart)

Shows memory usage for Jenkins nodes over time.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_node")
  |> filter(fn: (r) => r["_field"] == "memory_available" or r["_field"] == "memory_total")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> pivot(rowKey: ["_time", "node_name"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      _time: r._time,
      node_name: r.node_name,
      _value: (float(v: r.memory_total) - float(v: r.memory_available)) / float(v: r.memory_total) * 100.0
    }))
```

---

## 8. Build Results Distribution
**Visualization Type:** Pie chart

Shows the distribution of build results (success, failure, aborted, etc.).

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result")
  |> group(columns: ["_value"])
  |> count()
  |> group()
```

---

## 9. Currently Running Builds
**Visualization Type:** Stat

Shows the count of builds currently in progress (if tracked).

```flux
from(bucket: "pipelines")
  |> range(start: -5m)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result")
  |> filter(fn: (r) => r["_value"] == "RUNNING" or r["_value"] == "IN_PROGRESS")
  |> group()
  |> count()
```

---

## 10. Average Build Duration by Job
**Visualization Type:** Bar chart (Vertical)

Shows average build duration for each job.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "duration")
  |> group(columns: ["name"])
  |> mean()
  |> group()
  |> sort(columns: ["_value"], desc: true)
  |> map(fn: (r) => ({name: r.name, _value: r._value / 1000.0}))
```

---

## 11. Build Frequency Over Time
**Visualization Type:** Time series (Bar chart)

Shows the number of builds started per time period.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "number")
  |> aggregateWindow(every: 1h, fn: count, createEmpty: false)
```

---

## 12. Node Disk Space Available
**Visualization Type:** Gauge (Multi-gauge)

Shows available disk space percentage for each Jenkins node.

```flux
from(bucket: "pipelines")
  |> range(start: -5m)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_node")
  |> filter(fn: (r) => r["_field"] == "disk_available")
  |> last()
  |> group(columns: ["node_name"])
  |> map(fn: (r) => ({
      node_name: r.node_name,
      _value: float(v: r._value) / 1073741824.0
    }))
```

---

## 13. Job Execution Timeline
**Visualization Type:** Table (with time formatting)

Shows when each job was last executed and its result.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result" or r["_field"] == "duration" or r["_field"] == "number")
  |> pivot(rowKey: ["_time", "name"], columnKey: ["_field"], valueColumn: "_value")
  |> group(columns: ["name"])
  |> last()
  |> group()
  |> sort(columns: ["_time"], desc: true)
```

---

## 14. Build Duration Trend (Moving Average)
**Visualization Type:** Time series (Line chart with smooth)

Shows the moving average of build durations to identify performance trends.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "duration")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> movingAverage(n: 5)
  |> map(fn: (r) => ({r with _value: r._value / 1000.0}))
```

---

## 15. Node Response Time
**Visualization Type:** Time series (Line chart)

Shows Jenkins node response times over time.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_node")
  |> filter(fn: (r) => r["_field"] == "response_time")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

---

## 16. Current Average Memory Usage Across All Nodes
**Visualization Type:** Gauge

Shows the current mean memory usage percentage across all Jenkins nodes.

```flux
from(bucket: "pipelines")
  |> range(start: -5m)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_node")
  |> filter(fn: (r) => r["_field"] == "memory_available" or r["_field"] == "memory_total")
  |> last()
  |> pivot(rowKey: ["node_name"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      node_name: r.node_name,
      _value: (float(v: r.memory_total) - float(v: r.memory_available)) / float(v: r.memory_total) * 100.0
    }))
  |> mean()
  |> keep(columns: ["_value"])
```

---

## 17. Current Average Disk Usage Across All Nodes
**Visualization Type:** Gauge

Shows the current mean disk usage percentage across all Jenkins nodes.

```flux
from(bucket: "pipelines")
  |> range(start: -5m)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_node")
  |> filter(fn: (r) => r["_field"] == "disk_available" or r["_field"] == "disk_total")
  |> last()
  |> pivot(rowKey: ["node_name"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      node_name: r.node_name,
      _value: (float(v: r.disk_total) - float(v: r.disk_available)) / float(v: r.disk_total) * 100.0
    }))
  |> mean()
  |> keep(columns: ["_value"])
```

---

## 18. Current Average Executor Usage Across All Nodes
**Visualization Type:** Gauge

Shows the current percentage of executors in use across all Jenkins nodes.

```flux
from(bucket: "pipelines")
  |> range(start: -5m)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_node")
  |> filter(fn: (r) => r["_field"] == "busy_executors" or r["_field"] == "total_executors")
  |> last()
  |> pivot(rowKey: ["node_name"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      node_name: r.node_name,
      _value: if r.total_executors > 0 then float(v: r.busy_executors) / float(v: r.total_executors) * 100.0 else 0.0
    }))
  |> mean()
  |> keep(columns: ["_value"])
```

---

## 19. Node Architecture Distribution
**Visualization Type:** Pie chart

Shows the percentage distribution of different architectures across all Jenkins nodes.

```flux
from(bucket: "pipelines")
  |> range(start: -5m)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_node")
  |> filter(fn: (r) => r["_field"] == "memory_total")
  |> last()
  |> group(columns: ["arch"])
  |> count()
  |> group()
  |> rename(columns: {arch: "_value", _value: "count"})
```

---

## 20. Offline Nodes Percentage Over Time
**Visualization Type:** Time series (Line chart)

Shows the percentage of offline nodes over time with hourly data points.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_node")
  |> filter(fn: (r) => r["_field"] == "memory_total")
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> map(fn: (r) => ({r with _value: if r.status == "offline" then 1.0 else 0.0}))
  |> drop(columns: ["node_name", "arch", "disk_path", "temp_path"])
  |> group(columns: ["_time"])
  |> mean(column: "_value")
  |> duplicate(column: "_stop", as: "_time")
  |> map(fn: (r) => ({_time: r._time, _value: r._value * 100.0}))
```

---

## 21. Build Success Rate Over Time
**Visualization Type:** Time series (Line chart)

Shows the average success rate of builds over time with hourly intervals.

```flux
success = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> filter(fn: (r) => r["_value"] == 0)
  |> aggregateWindow(every: 1h, fn: count, createEmpty: true)
  |> set(key: "_field", value: "success")

total = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> aggregateWindow(every: 1h, fn: count, createEmpty: true)
  |> set(key: "_field", value: "total")

union(tables: [success, total])
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      _time: r._time,
      _value: if r.total > 0 then float(v: r.success) / float(v: r.total) * 100.0 else 0.0
    }))
```

---

## 22. Executor Usage Percentage Over Time
**Visualization Type:** Time series (Line chart)

Shows the percentage of executors in use across all nodes over time.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins")
  |> filter(fn: (r) => r["_field"] == "busy_executors" or r["_field"] == "total_executors")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      _time: r._time,
      _value: if r.total_executors > 0 then (r.busy_executors / r.total_executors) * 100.0 else 0.0
    }))
```

---

## 23. Build Results Distribution Over Time
**Visualization Type:** Time series (Line chart)

Shows the percentage of jobs with each result code over time. Use 5 separate queries (A-E) in Grafana to display all result codes on one graph.

**Query A - SUCCESS:**
```flux
success = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> filter(fn: (r) => r["_value"] == 0)
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> count()
  |> set(key: "_field", value: "success_count")

total = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> count()
  |> set(key: "_field", value: "total_count")

union(tables: [success, total])
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      _time: r._time,
      _value: if r.total_count > 0 then float(v: r.success_count) / float(v: r.total_count) * 100.0 else 0.0
    }))
  |> set(key: "_field", value: "SUCCESS")
  |> group()
```

**Query B - FAILURE:**
```flux
failure = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> filter(fn: (r) => r["_value"] == 1)
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> count()
  |> set(key: "_field", value: "failure_count")

total = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> count()
  |> set(key: "_field", value: "total_count")

union(tables: [failure, total])
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      _time: r._time,
      _value: if r.total_count > 0 then float(v: r.failure_count) / float(v: r.total_count) * 100.0 else 0.0
    }))
  |> set(key: "_field", value: "FAILURE")
  |> group()
```

**Query C - NOT_BUILD:**
```flux
notbuild = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> filter(fn: (r) => r["_value"] == 2)
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> count()
  |> set(key: "_field", value: "notbuild_count")

total = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> count()
  |> set(key: "_field", value: "total_count")

union(tables: [notbuild, total])
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      _time: r._time,
      _value: if r.total_count > 0 then float(v: r.notbuild_count) / float(v: r.total_count) * 100.0 else 0.0
    }))
  |> set(key: "_field", value: "NOT_BUILD")
  |> group()
```

**Query D - UNSTABLE:**
```flux
unstable = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> filter(fn: (r) => r["_value"] == 3)
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> count()
  |> set(key: "_field", value: "unstable_count")

total = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> count()
  |> set(key: "_field", value: "total_count")

union(tables: [unstable, total])
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      _time: r._time,
      _value: if r.total_count > 0 then float(v: r.unstable_count) / float(v: r.total_count) * 100.0 else 0.0
    }))
  |> set(key: "_field", value: "UNSTABLE")
  |> group()
```

**Query E - ABORTED:**
```flux
aborted = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> filter(fn: (r) => r["_value"] == 4)
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> count()
  |> set(key: "_field", value: "aborted_count")

total = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> count()
  |> set(key: "_field", value: "total_count")

union(tables: [aborted, total])
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({
      _time: r._time,
      _value: if r.total_count > 0 then float(v: r.aborted_count) / float(v: r.total_count) * 100.0 else 0.0
    }))
  |> set(key: "_field", value: "ABORTED")
  |> group()
```

**Note:** The above 5 queries can be combined into a single query:

```flux
// Count jobs per result code per hour
counts = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> duplicate(column: "_value", as: "result_code")
  |> group(columns: ["_time", "result_code"])
  |> count()

// Count total jobs per hour
total = from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "result_code")
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> count()
  |> rename(columns: {_value: "total"})

// Join and calculate percentages
join(tables: {counts: counts, total: total}, on: ["_time"])
  |> map(fn: (r) => ({
      _time: r._time,
      _field: if r.result_code == 0 then "SUCCESS"
              else if r.result_code == 1 then "FAILURE"
              else if r.result_code == 2 then "NOT_BUILD"
              else if r.result_code == 3 then "UNSTABLE"
              else if r.result_code == 4 then "ABORTED"
              else "UNKNOWN",
      _value: if r.total > 0 then float(v: r._value) / float(v: r.total) * 100.0 else 0.0
    }))
  |> group()
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
```

---

## 24. Average Build Duration Over Time (All Jobs)
**Visualization Type:** Time series (Line chart)

Shows the average build duration across all jobs over time in seconds.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "duration")
  |> group()
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> map(fn: (r) => ({_time: r._time, _value: r._value / 1000.0}))
```

---

## 25. Average Duration of Last Build of All Jobs
**Visualization Type:** Stat

Shows the average duration (in seconds) of the most recent build across all jobs.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "duration")
  |> group(columns: ["name"])
  |> last()
  |> group()
  |> mean()
  |> map(fn: (r) => ({_value: r._value / 1000.0}))
```

---

## 26. Number of Jobs Over Time
**Visualization Type:** Time series (Line chart)

Shows the count of unique jobs over time with 1-hour aggregation windows.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "number")
  |> aggregateWindow(every: 1h, fn: last, createEmpty: false)
  |> group(columns: ["_time"])
  |> distinct(column: "name")
  |> count()
```

---

## 27. Number of Builds Over Time
**Visualization Type:** Time series (Bar chart)

Shows the total count of builds executed per hour across all jobs.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "number")
  |> aggregateWindow(every: 1h, fn: count, createEmpty: false)
  |> group(columns: ["_time"])
  |> sum()
```

---

## 28. Cumulative Sum of Builds Over Time
**Visualization Type:** Time series (Line chart)

Shows the cumulative total of all builds executed over time.

```flux
from(bucket: "pipelines")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "jenkins_job")
  |> filter(fn: (r) => r["_field"] == "number")
  |> aggregateWindow(every: 1h, fn: count, createEmpty: false)
  |> group(columns: ["_time"])
  |> sum()
  |> cumulativeSum()
```

---

## Notes

- **Time Range Variables**: All queries use `v.timeRangeStart` and `v.timeRangeStop` which Grafana automatically provides based on the dashboard's time range selector.
- **Window Period**: `v.windowPeriod` is automatically calculated by Grafana based on the time range and panel width.
- **Result Codes**: Jenkins typically uses: 0 = SUCCESS, 1 = UNSTABLE, 2 = FAILURE
- **Units**: Build durations are in milliseconds in InfluxDB. Some queries divide by 1000 to convert to seconds, or 1073741824 to convert bytes to GB.
- **Grouping**: Use `group()` to reset grouping, `group(columns: ["field"])` to group by specific fields.

## Tips for Using in Grafana

1. **Configure InfluxDB Data Source** with Flux query language
2. **Set proper units** in panel settings (milliseconds, bytes, etc.)
3. **Use field overrides** to customize colors based on build results
4. **Add thresholds** to gauges for warning/critical levels
5. **Enable legends** on time series to identify different jobs
6. **Sort tables** by time descending to show most recent first
7. **Set refresh intervals** to match your Telegraf collection interval (120s)
