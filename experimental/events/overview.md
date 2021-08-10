# OpenTelemetry Events Overview

**Status**: [Experimental](../document-status.md)

* [Introduction](#introduction)
* [Limitations of non-OpenTelemetry Solutions](#limitations-of-non-opentelemetry-solutions)
* [OpenTelemetry Solution](#opentelemetry-solution)
* [Log Correlation](#log-correlation)
* [Events and Logs](#events-and-logs)
* [Legacy and Modern Log Sources](#legacy-and-modern-log-sources)
  * [System Logs](#system-logs)
  * [Infrastructure Logs](#infrastructure-logs)
  * [Third-party Application Logs](#third-party-application-logs)
  * [Legacy First-Party Applications Logs](#legacy-first-party-applications-logs)
  * [New First-Party Application Logs](#new-first-party-application-logs)
* [OpenTelemetry Collector](#opentelemetry-collector)
* [Auto-Instrumenting Existing Logging](#auto-instrumenting-existing-logging)
* [Trace Context in Legacy Formats](#trace-context-in-legacy-formats)

## Introduction

The OpenTelemetry standard has done a great job of identifying the use cases and
the model representing metrics, traces and logs.

In this document, we address events. Events are rich data points, documents
representing data either extracted from a system or the byproduct of an
operation. Examples of events are JSON payloads of REST API responses, SQL query
responses or CSV outputs.

This completes the vision of OpenTelemetry as a tool to collect and process
telemetry data from a variety of sources.

## Limitations of non-OpenTelemetry Solutions

Existing event collections solutions exist today, but require to be deployed
separately from other telemetry tooling. Events typically are unrelated to
tracing and monitoring tools, and thus no correlation exists between metrics,
traces, logs and events. Developers sometimes will attempt to represent rich 
event data in logs, at the peril of overloading logs and limiting the data
they are willing to send to files. 
There is no standardized way to include the information about the origin and
source of events (such as the application and the location/infrastructure where
the application runs) that is uniform with traces and metrics and allows all
telemetry data to be fully correlated in a precise and robust manner.

Similarly, events have no standardized way to propagate and record the request
execution context. In distributed systems this often results in a disjoint set
of events collected from different components of the system.

This is how a typical non-OpenTelemetry observability collection pipeline looks
like today:

![Separate Collection Diagram](img/separate-collection.png)

There are often different libraries and different collection agents, using
different protocols and data models, with telemetry data ending up in separate
backends that don't know how to work well together.

## OpenTelemetry Solution

Distributed tracing introduced the notion of request context propagation.

Fundamentally, though, nothing prevents the events to adopt the same context
propagation concepts. If the recorded events contained request context identifiers
(such as trace and span ids or user-defined baggage) it would result
in much richer correlation between events and traces, as well as correlation
between events emitted by different components of a distributed system.
This would make events valuable in distributed systems.

This is one of the promising evolutionary directions for observability tools.
Standardizing event correlation with logs, traces and metrics, adding support for
distributed context propagation for events, unification of source attribution of
events, traces and metrics will increase the individual and combined value of
observability information for legacy and modern systems. This is the vision of
OpenTelemetry's collection of events, traces and metrics:

![Unified Collection Diagram](img/unified-collection.png)

We emit events, logs, traces and metrics in a way that is compliant with 
OpenTelemetry data models, send the data through OpenTelemetry Collector, 
where it can be enriched and processed in a uniform manner. 
For example, Collector can add to all telemetry data coming from a Kubernetes 
Pod several attributes that describe the pod and it can be done automatically 
using [k8sprocessor](https://pkg.go.dev/github.com/open-telemetry/opentelemetry-collector-contrib/processor/k8sprocessor?tab=doc)
without the need for the Application to do anything special. Most importantly
such enrichment is completely uniform for all 3 signals. The Collector
guarantees that events, logs, traces and metrics have precisely the same 
attribute names and values describing the Kubernetes Pod that they come from.
This enables exact and unambiguous correlation of the signals by the Pod in the
 backend.

We decided to emit events with the most simple approach. One cannot know add a 
schema to events, as they may represent unstructured or schema-less data.

There are also countless existing prebuilt applications or systems that emit
events in certain formats. Operators of such applications have no or limited
control on how the logs are emitted. OpenTelemetry needs to support these events.

Given the above state of the logging space we took the following approach:

- OpenTelemetry defines an [event data model](data-model.md). The purpose of the
  data model is to have a common understanding of what an event record is, what
  data needs to be recorded, transferred, stored and interpreted by a logging
  system.

- Newly designed event systems are expected to emit events according to
  OpenTelemetry's event data model. More on this [later](#new-first-party-application-events).

- Existing event formats can be
  [unambiguously mapped](data-model.md#appendix-a-example-mappings) to
  OpenTelemetry event data model. OpenTelemetry Collector can read such events 
  and translate them to OpenTelemetry event data model.

- Existing applications can be modified to emit events according to 
  OpenTelemetry event data model. OpenTelemetry does not define a new
  eventing API that application developers are expected to call. Instead we opt
  to make it easy to continue using the common libraries that already
  exist. OpenTelemetry provides guidance on how applications or event
  libraries can be modified to become OpenTelemetry-compliant (link TBD). We
  also provide SDKs for some languages (link TBD) that make it easy to modify
  the existing event libraries so that they emit OpenTelemetry-compliant events.

This approach allows OpenTelemetry to read existing system and application logs,
provides a way for newly built application to emit rich, structured,
OpenTelemetry-compliant logs, and ensures that all logs are eventually
represented according to a uniform event data model on which the backends can
operate.

In the future OpenTelemetry may define a new eventing API and provide
implementations for various languages (like we currently do for logs, traces and
metrics), but it is not an immediate priority.

Later in this document we will discuss in more details
[how various event sources are handled](#legacy-and-modern-event-sources) by
OpenTelemetry, but first we need to describe in more details an important
concept: the event correlation.

## Event Correlation

Events can be correlated with the rest of observability data in a few dimensions:

- By the **time of execution**. Events, logs, traces and metrics can record the 
  moment of time or the range of time the execution took place. This is the 
  most basic form of correlation.

- By the **execution context**, also known as the request context. It is a
  standard practice to record the execution context (trace and span ids as well
  as user-defined context) in the spans. OpenTelemetry extends this practice to
  events where possible by including [TraceId](data-model.md#field-traceid) and
  [SpanId](data-model.md#field-spanid) in the event records. This allows to
  directly correlate events and traces that correspond to the same execution
  context. It also allows to correlate events from different components of a
  distributed system that participated in the particular request execution.

- By the **origin of the telemetry**, also known as the Resource context.
  OpenTelemetry traces and metrics contain information about the Resource they
  come from. We extend this practice to events by including the
  [Resource](data-model.md#field-resource) in event records.
  
These 3 correlations can be the foundation of powerful navigational, filtering,
querying and analytical capabilities. OpenTelemetry aims to record and collects
events in a manner that enables such correlations.

## Legacy and Modern Event Sources

It is important to distinguish several sorts of legacy and modern event sources.
Firstly, this directly affects how exactly we get access to these events and how
we collect them. Secondly, we have varying levels of control over how these events
are generated and whether we can amend the information that can be included in
the events.

Below we list several categories of events and describe what can be possibly done
for each category to have better experience in the observability solutions.

### Polling

These are events generated by polling regularly a REST endpoint or a data store,
 such as MySQL, PostgreSQL or MongoDB.
Examples of REST polling format are JSON plain text, or CSV events.

OpenTelemetry Collector can ingest REST polling events (link TBD), database 
polling events (link TBD) and automatically enrich them with Resource 
information using the [resourcedetection](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/master/processor/resourcedetectionprocessor)
processor.

### Transaction event data

These are events generated as part of the execution of a transactional system,
representing a change of state, an order or a message.

OpenTelemetry Collector or other agents can be used to ingest events from most
common message brokers or transactional systems.

### Third-party Application Events

Applications typically write events to logs, to files or other
specialized medium (e.g. Windows Event Logs for applications). These events can
be in many different formats, spanning a spectrum along these variations:

- Free-form text formats with no easily automatable and reliable way to parse
  structured data from them.

- Better specified and sometimes customizable formats that can be parsed to
  extract structured data (such as JSON or XML).


The collection system needs to be able to discover most commonly used
applications and have parsers that can convert these events into a structured
format. Like system and infrastructure events, application events often lack request
context but can be enriched by resource context, including the attributes that
describe the host and infrastructure as well as application-level attributes
(such as the application name, version, name of the database - if it is a DBMS,
etc).

OpenTelemetry recommends to collect application events using Collector's
[filelog receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver).
Alternatively, another event collection agent, such as HEC, can collect
events, then send
to OpenTelemetry Collector where the events can be further processed and enriched.

### Legacy First-Party Applications Events

These are applications that are created in-house. People tasked with setting up
event collection infrastructure sometimes are able to modify these applications to
alter how events are written and what information is included in the events. For
example, the application’s logic may be changed to send events to a collector
instead of writing to a flat file or a database.

More significant modifications to these applications can be done manually by
their developers, such as addition of the request context to every event, 
however this is likely going to be vanishingly rare due to the effort
required.

As opposed to manual efforts we have an interesting opportunity to "upgrade"
application events in a less laborious way by providing full or semi
auto-instrumenting solutions that modify the application to automatically output
 the request context such as the trace id or span id with every event emission.
  The request context can be automatically
extracted from incoming requests if standard compliant request propagation is
used, e.g. via [W3C TraceContext](https://w3c.github.io/trace-context/). In
addition, the requests outgoing from the application may be injected with the
same request context data, thus resulting in context propagation through the
application and creating an opportunity to have full request context in events
collected from all applications that can be instrumented in this manner.

### New First-Party Application Events

These are greenfield developments. OpenTelemetry provides recommendations and
best practices about how to emit events (along with logs, traces and metrics) 
from these applications. For applicable languages and frameworks the 
auto-instrumentation or simple configuration of an eventing library to use an 
OpenTelemetry appender or extension will still be the easiest way to emit 
context-enriched events. The extensions
will support the inclusion of the request context in the events and allow to send
events using OTLP protocol to the backend or to the Collector, bypassing the need
to have the events represented as text files or database tables. Emitted events
 are automatically augmented by application-specific resource context 
 (e.g. process id, programming language, event library name and version, etc). 
 Full correlation across all context dimensions will be available for these events.

OpenTelemetry defines a new eventing API and creates new user-facing eventing 
libraries. Our initial goal is to simplify event to send enriched, contextual 
events. This is how a typical new application uses the OpenTelemetry API:

TODO image?

## OpenTelemetry Collector

To enable event collection according to this specification we use OpenTelemetry
Collector.

The following functionality exists to enable event collection:

- Support for event data type and event pipelines based on the
  [event data model](data-model.md). This includes processors such as
  [attributesprocessor](https://github.com/open-telemetry/opentelemetry-collector/tree/main/processor/attributesprocessor)
  that can operate on event data.

- Ability to receive events via common network protocols for events, such as 
  HTTP polling and interpret them according to semantic conventions defined in this
  specification.

- Ability to send events via common network protocols for events, such as HEC, or
  vendor-specific event formats. Collector contains exporters that directly
  implement this ability.

## Trace Context in Legacy Formats

Earlier we briefly mentioned that it is possible to modify existing applications
so that they include the Request Context information in the emitted events.

[OTEP0114](https://github.com/open-telemetry/oteps/pull/114) defines how the
trace context should be recorded in logs. We apply the same logic to events.
To summarize, the following field names should be used in legacy formats:

- "trace_id" for [TraceId](data-model.md#field-traceid), hex-encoded.
- "span_id" for [SpanId](data-model.md#field-spanid), hex-encoded.
- "trace_flags" for [trace flags](data-model.md#field-traceflags), formatted
  according to W3C traceflags format.

All 3 fields are optional (see the [data model](data-model.md) for details of
which combination of fields is considered valid).
