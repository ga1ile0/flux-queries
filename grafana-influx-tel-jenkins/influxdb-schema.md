# InfluxDB Schema Documentation

This document describes all measurements, tags, and fields pushed to the `pipelines` bucket in InfluxDB.

---

## Measurements

### jenkins_job
Metrics for Jenkins job builds, collected every 4 hours.

#### Tags
- **name** - The name of the Jenkins job
- **parents** - Parent folder path for the job
- **result** - Build result status (SUCCESS, FAILURE, UNSTABLE, ABORTED, etc.)
- **port** - Jenkins server port
- **source** - Source identifier (typically "jenkins")

#### Fields
- **duration** - Total build execution time in milliseconds
- **result_code** - Numeric result code (0 = SUCCESS, 1 = FAILURE, 2 = NOT_BUILD, 3 = UNSTABLE, 4 = ABORTED)
- **number** - Build number for this job
- **total_duration** - Total duration including queue time in milliseconds

---

### jenkins_node
Metrics for Jenkins node/agent health and resources, collected every 120 seconds via HTTP API.

#### Tags
- **node_name** - Display name of the Jenkins node
- **arch** - System architecture of the node (e.g., Linux amd64)
- **disk_path** - File system path monitored for disk space
- **temp_path** - File system path for temporary space
- **status** - Node status ("online" or "offline")

#### Fields
- **memory_available** - Available physical memory in bytes
- **memory_total** - Total physical memory in bytes
- **swap_available** - Available swap space in bytes
- **swap_total** - Total swap space in bytes
- **disk_available** - Available disk space in bytes at disk_path
- **temp_available** - Available temporary space in bytes at temp_path
- **response_time** - Node response time in milliseconds
- **busy_executors** - Number of executors currently running builds
- **total_executors** - Total number of executors on the node

---

## Notes

- All byte values can be converted to GB by dividing by 1073741824
- All millisecond values can be converted to seconds by dividing by 1000
- Timestamps are stored in nanosecond precision
- The `jenkins_node` measurement fields are converted to integers via the converter processor
