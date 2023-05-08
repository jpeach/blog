---
title: "Metrics data model notes"
date: 2017-07-24T13:19:38+10:00
tags: [metrics]
featured_image: ""
description: ""
---

Some notes on the data models of various metrics collection systems.

## Performance Co-Pilot 

[Performance Co-Pilot](http://pcp.io) is a metrics collection and
visualization system heavily inspired by SNMP. PCP originates in
the systems monitoring world.

PCP has a fairly rich vocabulary to [describe
metrics](http://pcp.io/books/PCP_PG/html/LE11914-PARENT.html)
according to their type, semantics, dimensions and scale.

* **type** is the fundamental data type of the metric, eg. string,
  uint32, uint64, double, binary

* **semantics** describe the logical behaviour of a metric and can
  be _counter_, _instant_ or _discrete_. Counters are increment-only
  metrics. Instantaneous metrics may increment or decrement but
  they vary over a continuous domain of values. Discrete metrics
  vary in arbitrary ways but don't represent a continuous value
  domain. For example, the total number of requests would be a
  counter, the number of currently executing requests would be
  instantaneous but the latency of the most recent request would
  be discrete.

* **dimensionality** describes the resource dimension of the
  metrics; either space, time or count.

* **scale** describes the the units of the metric dimension. For
  example a time dimension metric could be published in seconds or
  hours, and a space metric could be bytes or megabytes. The scale
  allows exporter to avoid datatype overflow and lets visualizers
  automatically rescale graphs to appropriate units.

Performance Metrics Descriptions
  - http://pcp.io/books/PCP_PG/html/LE11914-PARENT.html

Semantics, Dimensions, Scale
  - http://pcp.io/books/PCP_UAG/html/id5189172.html

Semantics
  - http://pcp.io/books/PCP_PG/html/id5190312.html

The PCP data model explicitly recognizes a single namespace dimension
for metrics, the instance domain. This allows a single metric to
have multiple named instances. The common uses case for this is for
resources like CPU and Disk, where systems commonly have more than
1 of each kind of resource. There is no way to explicitly model
multiple namespace dimensions so users typically resort to naming
conventions and workarounds.

## Graphite

[Graphite](https://graphiteapp.org/) is a metrics storage and
visualization system.

The Graphite [data model](http://graphite.readthedocs.io/en/latest/feeding-carbon.html)
is extremely simple, consisting of just a
metric name and a value. The data model itself doesn't distinguish
between counters and instantaneous values, relying on operators to
apply the appropriate conversion functions when creating graphs.

In general, users of graphite establish metric naming conventions
to construct hierarchy and to represent namespace dimensionality.
Units and metric documentation are also typically addresses by
judicious metric nomenclature.

## Prometheus

[Prometheus](https://prometheus.io/) is a metrics collection and
visualization system with roots in both the systems and applications
monitoring world. It seems quite heavily inspired by Graphite; it
has a richer data model, though it still relies on metric naming
conventions for some aspects of the data model.

The Prometheus data format allows the specification of metric
documentation and type.

The fundamental metric types in the Prometheus data model are
counters and gauges (instantaneous values). There is no explicit
specification of the underlying data type, though protobuf exposition
format specifies double in both cases.

Prometheus has 2 types of aggregate metrics, "summary" and "histogram".
Both of these metric types include a sum (total of observed
measurements), a count (of observations) and a set of histogram
buckets. Summary metrics are intended for publishing pre-calculated
quantiles, so are closely related to timed observations of request
processing. Histograms are more general, with more arbitrary
interpretation of buckets.

Data model
  - https://prometheus.io/docs/concepts/data_model/

Wire format
  - https://prometheus.io/docs/instrumenting/exposition_formats/

Comparisons
  - https://prometheus.io/docs/practices/histograms/
  - https://prometheus.io/docs/introduction/comparison/  

Prometheus explicitly supports arbitrary namespace dimensionality
by allowing publishers to specify label pairs on metrics. Each
unique set of label pairs implicitly creates a unique time series.

## Circonus

[Circonus](https://www.circonus.com) is a complete monitoring
infrastructure that includes data collection, metrics storage,
graphing UI and alerting. Unlike other systems here, Circonus is
designed to ingest metrics in a number of different serialization
formats, including HTTP JSON checks, statsd, resmon, and others.

The Circonus [data
model](https://login.circonus.com/resources/docs/user/Data.html#Data)
consists of numbers, strings and histograms. For numeric data,
Circonus doesn't distinguish and particular semantics, but always
generates derived metrics that would be appropriate to both counters
and gauges. Units and tags can be associated with a metric that is
present in the data store.

Circonus applies [heat
map](https://login.circonus.com/resources/docs/user/Visualization/Graphs/View/Histograms.html#Histograms)
visualization to histogram data. It is not clear whether you need
to clear the histogram bins between samples or whether Circonus
will do the relevant arithmetic automatically (probably the
    latter?).

## Dropwizard

[Dropwizard](http://metrics.dropwizard.io/) is a Java library for
metrics instrumentation. It provides a number of abstractions for
collecting application metrics, and for publishing them to various
collection systems.

The core Dropwizard
[Metrics](http://metrics.dropwizard.io/3.2.3/manual/core.html) types
are Counter and Gauge, however these types represent implementation
choices not metrics semantics as we see in other systems. For
example, a Dropwizard Counter is allowed to be decremented, which
violates counting semantics.

Dropwizard metrics do not have a notion of metric units and do not
publish any per-metric documentation.

Compared to the Mesos implementation, Dropwizard has a richer set
of Gauge metrics, including derived (based on the values from other
Gauges) and cached (rate-limited) Gauges.

Dropwizard supports the concept of "reporters", which are adaptors
that can present the Dropwizard metrics collection in a number fo
different formats, eg. as JMX objects or a Graphite metrics stream.
