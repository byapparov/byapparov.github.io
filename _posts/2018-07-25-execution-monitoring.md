---
layout: post
title: "Execution Monitoring for R Scripts"
date: 2018-07-25 16:16:01 -0600
categories: [post, package]
tags: [package, rdqa, r]
---

If you have automated your data processing scripts to run as cron job you need a monitoring and alerting system.  

You might also need some mechanism to do incremental data processing to reduce the workload and make solution scalable.

[InfluxDb](https://www.influxdata.com/) and [Grafana](https://grafana.com/) provide a convenient way of monitoring and alerting.

You can write execution summary into InfluxDb for each job and build monitoring for your jobs in Grafana which has native integration with Influx.

## Working with Metadata
For our automated data pipeline jobs I wrote [rmeta](https://github.com/byapparov/rmeta) - a small wrapper package over [influxdbr](https://github.com/dleutnant/influxdbr) which allows to write two types of metrics into Influx:

* executions - represents execution of a batch data pipeline job
* task - represents a single task in the data pipeline job

### Execution

| field         |   type        | sample values                        |
|:-------------:|:-------------:|:-------------------------------------|
| time          | timestamp     | 2015-08-18T00:06:00Z                 |
| id            | tag           | d1b5ece8-075d-4448-a0a4-465e9e89644c |
| job           | tag           | customer_pipeline                    |
| state         | tag           | start, end, error                    |
| value         | field         | 1                                    |

### Task

| field         |   type        | sample values        |
|:-------------:|:-------------:|:---------------------|
| time          | timestamp     | 2015-08-18T00:06:00Z |
| id            | tag           | d1b5ece8-075d-4448-a0a4-465e9e89644c |
| job           | tag           | customer_pipeline    |
| type          | tag           | load                 |
| datasource    | tag           | events_table         |
| records       | field (int)   | 1000                 |
| increment     | field (int)   | 100001               |


Integration of the metadata logging into the database with `rmeta` is simple.

First, call `start_job()` which logs execution of `start` type into `execution` metric. It will also save name of the job in the `rmeta` package environment. Name of the job will be used in the calls of any other function until `end_job()` is called.

To implement incremental load you can read last increment (should be a strictly monotonically increasing integer field) with `read_increment()`. Once processing of the delta is complete you can log new increment with `log_load`.

Here is an example of the script that uses `rmeta`:

```R
start_job("my_pipeline")

# find where we finished the last time
target_data.increment <- read_increment("target_table")

# use increment to load delta (new data since the last execution)
dt <- loadDataFunction(target_data.increment)

# pre-processes data and get (dt)
dt <- prepareDataFunction(dt)

target_data.new_increment <- max(dt$increment_integer_field)
target_data.records <- nrow(dt)

# save new increment for the next delta load
log_load(
  destination = "target_table",
  records = target_data.records,
  increment = target_data.new_increment
)

# complete the execution
end_job()
```

This code would create the following records:

**execution**

|time           |id             | job              |state | value |
|:-------------:|:-------------:|:-----------------|:-----|-------|
|2015-08-18T00:06:00Z|d1b5ece8-075d-4448-a0a4-465e9e89644c|my_pipeline|start|1|
|2015-08-18T00:06:30Z|d1b5ece8-075d-4448-a0a4-465e9e89644c|my_pipeline|end  |1|

**task**

|time                |id                                  | job       |type | datasource |records|increment|
|:------------------:|:----------------------------------:|:----------|:----|:-----------|-------|---------|
|2015-08-18T00:06:00Z|d1b5ece8-075d-4448-a0a4-465e9e89644c|my_pipeline|load |target_table|1000   | 10001   |


## Monitoring and Alerts

To set up monitoring in [Grafana](http://docs.grafana.org/guides/getting_started/) create new dashboard and add a graph that [links to your Influx database](http://docs.grafana.org/features/datasources/influxdb/). You can build a query that is getting `sum(value)` from `execution` measurement where `state="end"` and `job="my_pipeline"`

Alerting system in Grafana allows you to send messages via different channels based on wide range of time based criteria.

For example if you can check that job is executed at least once in last 24 hr and send an alert to Slack if it was not.
