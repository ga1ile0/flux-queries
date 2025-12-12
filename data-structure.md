**Below is the structure of the data in Influx**

Measurements:
1. jenkins
2. jenkins_node
3. jenkins_job

Description of each measuerement:

1. jenkins
tags:
- source
- port
fields:
- busy_executors
- total_executors

2. jenkins_node
tags:
- arch
- disk_path
- temp_path
- node_name
- status (“online”, “offline”)
- source
- port
fields:
- disk_available (Bytes)
- temp_available (Bytes)
- memory_available (Bytes)
- memory_total (Bytes)
- swap_available (Bytes)
- swap_total (Bytes)
- response_time (ms)
- num_executors

3. jenkins_job
tags:
- name
- parents
- result
- source
- port
fields:
- duration (ms)
- number
- result_code (0 = SUCCESS, 1 = FAILURE, 2 = NOT_BUILD, 3 = UNSTABLE, 4 = ABORTED)