---
title: "Prometheus Primer"
date: 2019-07-04T14:56:12-04:00
draft: false
toc: true
tags:
  - prometheus
---

# Querying Basics

Queries run against *metrics*, which are sets of timeseries data. They have millisecond granularity and are stored as floating point values.

# Using Queries

Queries reference individual metrics and perform some analysis on them. Most often you use the `rate` function to "bucket" a metric into time intervals. Once the metric in question has been bucketed into time intervals, you can do comparisons.

```
(rate(http_response_size_bytes[1m])) > 512
```

This query takes the size of http responses in bytes and buckets it into one minute intervals and drops any data points smaller than 512 bytes. Variations on this query could be used to analyse how bandwidth is being consumed across your instrumented processes; a spike or trending rise in high bandwidth requests could trigger an alert to prevent data overages breaking the bank.


```
sum without(instance, node_name, hostname, kubernetes_io_hostname) (rate(http_request_duration_microseconds[1m])) > 2000
```

This query looks at the metric `http_request_duration_microseconds`, buckets it into one minute intervals, and then drops all data points that are smaller than 2000 microseconds. Increases in response durations might indicate network congestion or other I/O contention.

## Labels

Prometheus lets you apply labels to your metrics. Some are specificed in the scrape configurations; these are usually things like the hostname of the machine, its datacenter or geographic region, etc. Instrumented applications can also specify labels when generating metrics; these are used to indicate things known at runtime like the specific HTTP route ( e.g. `/blog` or `/images/kittens` ) being measured.

Prometheus queries allow you to specify labels to match against which will let you control how your data is grouped together; you can query against geographic regions, specific hostnames, etc. It also supports regular expressions so you can match against patterns instead of literal strict matches.

```
(rate(http_response_size_bytes{kubernetes_io_hostname="node-y3ul"}[1m])) > 512
(rate(http_response_size_bytes{version=~"v1\.2\.*"}[1m])) > 512
```

An important consideration is that when querying, prometheus considers metrics with any difference in labels as distinct sets of data. Two HTTP servers running in the same datacenter can have different hostnames in their labels; this is useful when you want to monitor error rates per-container but can be detrimental when you want to examine the data for the datacenter as a whole.

To that end, prometheus gives you the ability to strip labels off the metrics in the context of a given query. This is useful for generating aggregate reports.

```
sum without(instance, node_name, hostname, kubernetes_io_hostname)(rate(go_goroutines[1m]))
```

# Alerts

All of this is fun to play with, but none of it is useful if you have to manually run the queries all the time. On its own, prometheus can generate "alerts" but these don't go anywhere on their own; they're set in the config file and look like this:

```
groups:
- name: example
  rules:
  - alert: HighErrorRate
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
    for: 10m
    labels:
      severity: page
    annotations:
      summary: High request latency
  - alert: TotalSystemFailure
    expr: job:avg_over_time(up{job="appName"}[5m]) < .5
    for: 5m
    labels:
      severity: page
    annotations:
      summary: Large scale application outage
```

Alerts can have labels and metadata applied much like regular data sources. On their own, however, they don't *do* anything. Fortunately, the prometheus team has released [AlertManager](https://github.com/prometheus/alertmanager) to work with these alerts. AlertManager receives these events and dispatches them to various services, ranging from email to slack channels to VictorOps or other paging services.

AlertManager lets you define teams and hierarchies that alerts can cascade through and create conditions during which some subsets of alerts are emporarily muted; if a higher priority event is breaking, more trivial alerts can be ignored for a short time if desired.
