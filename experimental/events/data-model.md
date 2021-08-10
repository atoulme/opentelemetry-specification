# Log Data Model

**Status**: [Experimental](../document-status.md)

* [Design Notes](#design-notes)
  * [Requirements](#requirements)
  * [Field Kinds](#field-kinds)
* [Log and Event Record Definition](#log-and-event-record-definition)
  * [Field: `Timestamp`](#field-timestamp)
  * [Trace Context Fields](#trace-context-fields)
    * [Field: `TraceId`](#field-traceid)
    * [Field: `SpanId`](#field-spanid)
    * [Field: `TraceFlags`](#field-traceflags)
  * [Severity Fields](#severity-fields)
    * [Field: `SeverityText`](#field-severitytext)
    * [Field: `SeverityNumber`](#field-severitynumber)
    * [Mapping of `SeverityNumber`](#mapping-of-severitynumber)
    * [Reverse Mapping](#reverse-mapping)
    * [Error Semantics](#error-semantics)
    * [Displaying Severity](#displaying-severity)
    * [Comparing Severity](#comparing-severity)
  * [Field: `Name`](#field-name)
  * [Field: `Body`](#field-body)
  * [Field: `Resource`](#field-resource)
  * [Field: `Attributes`](#field-attributes)
* [Example Log Records](#example-log-records)
* [Appendix A. Example Mappings](#appendix-a-example-mappings)
  * [RFC5424 Syslog](#rfc5424-syslog)
  * [Windows Event Log](#windows-event-log)
  * [SignalFx Events](#signalfx-events)
  * [Splunk HEC](#splunk-hec)
  * [Log4j](#log4j)
  * [Zap](#zap)
  * [Apache HTTP Server access log](#apache-http-server-access-log)
  * [CloudTrail Log Event](#cloudtrail-log-event)
  * [Google Cloud Logging](#google-cloud-logging)
* [Elastic Common Schema](#elastic-common-schema)
* [Appendix B: `SeverityNumber` example mappings](#appendix-b-severitynumber-example-mappings)
* [References](#references)

This is a data model and semantic conventions that allow to represent events from
various sources: polling, messages, transactions, etc. Existing event formats 
can be unambiguously mapped to this data model.
Reverse mapping from this data model is also possible to the extent that the
target log format has equivalent capabilities.

The purpose of the data model is to have a common understanding of what an event
record is, what data needs to be recorded, transferred, stored and interpreted
by a logging system.

This proposal defines a data model for
[Standalone Events](../glossary.md#standalone-event).

## Design Notes

### Requirements

The Data Model was designed to satisfy the following requirements:

- It should be possible to unambiguously map existing event formats to this Data
  Model. Translating log data from an arbitrary event format to this Data Model
  and back should ideally result in identical data.

- Mappings of other event formats to this Data Model should be semantically
  meaningful. The Data Model must preserve the semantics of particular elements
  of existing event formats.

- Translating event data from an arbitrary event format A to this Data Model and
  then translating from the Data Model to another event format B ideally must
  result in a meaningful translation of event data that is no worse than a
  reasonable direct translation from event format A to event format B.

- It should be possible to efficiently represent the Data Model in concrete
  implementations that require the data to be stored or transmitted. We
  primarily care about 2 aspects of efficiency: CPU usage for
  serialization/deserialization and space requirements in serialized form. This
  is an indirect requirement that is affected by the specific representation of
  the Data Model rather than the Data Model itself, but is still useful to keep
  in mind.

### Definitions Used in this Document

In this document we refer to types `any` and `map<string, any>`, defined as
follows.

#### Type `any`

Value of type `any` can be one of the following:

- A scalar value: number, string or boolean,

- A byte array,

- An array (a list) of `any` values,

- A `map<string, any>`.

#### Type `map<string, any>`

Value of type `map<string, any>` is a map of string keys to `any` values. The
keys in the map are unique (duplicate keys are not allowed). The representation
of the map is language-dependent.

Arbitrary deep nesting of values for arrays and maps is allowed (essentially
allows to represent an equivalent of a JSON object).

### Field Kinds

This Data Model defines a logical model for a log record (irrespective of the
physical format and encoding of the record). Each record contains 2 kinds of
fields:

- Named top-level fields of specific type and meaning.

- Fields stored as `map<string, any>`, which can contain arbitrary values of
  different types. The keys and values for well-known fields follow semantic
  conventions for key names and possible values that allow all parties that work
  with the field to have the same interpretation of the data. See references to
  semantic conventions for `Resource` and `Attributes` fields and examples in
  [Appendix A](#appendix-a-example-mappings).

The reasons for having these 2 kinds of fields are:

- Ability to efficiently represent named top-level fields, which are almost
  always present (e.g. when using encodings like Protocol Buffers where fields
  are enumerated but not named on the wire).
  
- Ability to enforce types of named fields, which is very useful for compiled
  languages with type checks.

- Flexibility to represent less frequent data as `map<string, any>`. This
  includes well-known data that has standardized semantics as well as arbitrary
  custom data that the application may want to include in the logs.

When designing this data model we followed the following reasoning to make a
decision about when to use a top-level named field:

- The field needs to be either mandatory for all records or be frequently
  present in well-known log and event formats (such as `Timestamp`) or is
  expected to be often present in log records in upcoming logging systems (such
  as `TraceId`).

- The field’s semantics must be the same for all known log and event formats and
  can be mapped directly and unambiguously to this data model.
  
Both of the above conditions were required to give the field a place in the
top-level structure of the record.

## Event Record Definition

[Appendix A](#appendix-a-example-mappings) contains many examples that show how
existing log formats map to the fields defined below. If there are questions
about the meaning of the field reviewing the examples may be helpful.

Here is the list of fields in a log record:

Field Name     |Description
---------------|--------------------------------------------
Timestamp      |Time when the event occurred.
TraceId        |Request trace id.
SpanId         |Request span id.
TraceFlags     |W3C trace flag.
Body           |The body of the event.
Resource       |Describes the source of the log.
Attributes     |Additional information about the event.

Below is the detailed description of each field.

### Field: `Timestamp`

Type: Timestamp, uint64 nanoseconds since Unix epoch.

Description: Time when the event occurred measured by the origin clock. This
field is optional, it may be missing if the timestamp is unknown.

### Trace Context Fields

#### Field: `TraceId`

Type: byte sequence.

Description: Request trace id as defined in
[W3C Trace Context](https://www.w3.org/TR/trace-context/#trace-id). Can be set
for logs that are part of request processing and have an assigned trace id. This
field is optional.

#### Field: `SpanId`

Type: byte sequence.

Description: Span id. Can be set for logs that are part of a particular
processing span. If SpanId is present TraceId SHOULD be also present. This field
is optional.

#### Field: `TraceFlags`

Type: byte.

Description: Trace flag as defined in
[W3C Trace Context](https://www.w3.org/TR/trace-context/#trace-flags)
specification. At the time of writing the specification defines one flag - the
SAMPLED flag. This field is optional.

### Field: `Body`

Type: any.

Description: A value containing the body of the event record (see the description
of `any` type above). Can be for example a human-readable string message
(including multi-line) describing the event in a free form or it can be a
structured data composed of arrays and maps of other values. Can vary for each
occurrence of the event coming from the same source.

### Field: `Resource`

Type: `map<string, any>`.

Description: Describes the source of the log, aka
[resource](../overview.md#resources). Multiple occurrences of events coming from
the same event source can happen across time and they all have the same value of
`Resource`. Can contain for example information about the application that emits
the record or about the infrastructure where the application runs. Data formats
that represent this data model may be designed in a manner that allows the
`Resource` field to be recorded only once per batch of log records that come
from the same source. SHOULD follow OpenTelemetry
[semantic conventions for Resources](../resource/semantic_conventions/README.md).
This field is optional.

### Field: `Attributes`

Type: `map<string, any>`.

Description: Additional information about the specific event occurrence. Unlike
the `Resource` field, which is fixed for a particular source, `Attributes` can
vary for each occurrence of the event coming from the same source. Can contain
information about the request context (other than TraceId/SpanId). SHOULD follow
OpenTelemetry
[semantic conventions for Attributes](../trace/semantic_conventions/README.md).
This field is optional.

## Example Event Records

Below are examples that show one possible representation of event records in JSON.
These are just examples to help understand the data model. Don’t treat the
examples as _the_ way to represent this data model in JSON.

This document does not define the actual encoding and format of the event record
representation. Format definitions will be done in separate OTEPs (e.g. the event
records may be represented as msgpack, JSON, Protocol Buffer messages, etc).

Example 1

```javascript
{
  "Timestamp": 1586960586000, // JSON needs to make a decision about
                              // how to represent nanoseconds.
  "Attributes": {
    "http.status_code": 500,
    "http.url": "http://example.com",
    "my.custom.application.tag": "hello",
  },
  "Resource": {
    "service.name": "donut_shop",
    "service.version": "2.0.0",
    "k8s.pod.uid": "1138528c-c36e-11e9-a1a7-42010a800198",
  },
  "TraceId": "f4dbb3edd765f620", // this is a byte sequence
                                 // (hex-encoded in JSON)
  "SpanId": "43222c2d51a7abe3",
  "Body": {"message": "I like donuts"}
}
```

Example 2

```javascript
{
  "Timestamp": 1586960586000,
  ...
  "Body": {
    "i": "am",
    "an": "event",
    "of": {
      "some": "complexity"
    }
  }
}
```

Example 3

```javascript
{
   "Timestamp": 1586960586000,
   "Attributes":{
      "http.scheme":"https",
      "http.host":"donut.mycie.com",
      "http.target":"/order",
      "http.method":"post",
      "http.status_code":500,
      "http.flavor":"1.1",
      "http.user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.149 Safari/537.36",
   }
}
```

## Appendix A. Example Mappings

This section contains examples of mapping of other event formats to
this data model.


### SignalFx Events

<table>
  <tr>
    <td>Field</td>
    <td>Type</td>
    <td>Description</td>
    <td>Maps to Unified Model Field</td>
  </tr>
  <tr>
    <td>Timestamp</td>
    <td>Timestamp</td>
    <td>Time when the event occurred measured by the origin clock.</td>
    <td>Timestamp</td>
  </tr>
  <tr>
    <td>EventType</td>
    <td>string</td>
    <td>Short machine understandable string describing the event type. SignalFx specific concept. Non-namespaced. Example: k8s Event Reason field.</td>
    <td>Name</td>
  </tr>
  <tr>
    <td>Category</td>
    <td>enum</td>
    <td>Describes where the event originated and why. SignalFx specific concept. Example: AGENT. </td>
    <td>Attributes["com.splunk.signalfx.event_category"]</td>
  </tr>
  <tr>
    <td>Dimensions</td>
    <td>map&lt;string, string></td>
    <td>Helps to define the identity of the event source together with EventType and Category. Multiple occurrences of events coming from the same event source can happen across time and they all have the value of Dimensions. </td>
    <td>Resource</td>
  </tr>
  <tr>
    <td>Properties</td>
    <td>map&lt;string, any></td>
    <td>Additional information about the specific event occurrence. Unlike Dimensions which are fixed for a particular event source, Properties can have different values for each occurrence of the event coming from the same event source.</td>
    <td>Attributes</td>
  </tr>
</table>

### Splunk HEC

<table>
  <tr>
    <td>Field</td>
    <td>Type</td>
    <td>Description</td>
    <td>Maps to Unified Model Field</td>
  </tr>
  <tr>
    <td>time</td>
    <td>numeric, string</td>
    <td>The event time in epoch time format, in seconds.</td>
    <td>Timestamp</td>
  </tr>
  <tr>
    <td>host</td>
    <td>string</td>
    <td>The host value to assign to the event data. This is typically the host name of the client that you are sending data from.</td>
    <td>Resource["host.hostname"]</td>
  </tr>
  <tr>
    <td>source</td>
    <td>string</td>
    <td>The source value to assign to the event data. For example, if you are sending data from an app you are developing, you could set this key to the name of the app.</td>
    <td>Resource["service.name"]</td>
  </tr>
  <tr>
    <td>sourcetype</td>
    <td>string</td>
    <td>The sourcetype value to assign to the event data.</td>
    <td>Attributes["source.type"]</td>
  </tr>
  <tr>
    <td>event</td>
    <td>any</td>
    <td>The JSON representation of the raw body of the event. It can be a string, number, string array, number array, JSON object, or a JSON array.</td>
    <td>Body</td>
  </tr>
  <tr>
    <td>fields</td>
    <td>map&lt;string, any></td>
    <td>Specifies a JSON object that contains explicit custom fields.</td>
    <td>Attributes</td>
  </tr>
  <tr>
    <td>index</td>
    <td>string</td>
    <td>The name of the index by which the event data is to be indexed. The index you specify here must be within the list of allowed indexes if the token has the indexes parameter set.</td>
    <td>TBD, most like will go to attributes</td>
  </tr>
</table>

## Elastic Common Schema

<table>
  <tr>
    <td>Field</td>
    <td>Type</td>
    <td>Description</td>
    <td>Maps to Unified Model Field</td>
  </tr>
  <tr>
    <td>@timestamp</td>
    <td>datetime</td>
    <td>Time the event was recorded</td>
    <td>Timestamp</td>
  </tr>
  <tr>
    <td>message</td>
    <td>string</td>
    <td>Any type of message</td>
    <td>Body</td>
  </tr>
  <tr>
    <td>labels</td>
    <td>key/value</td>
    <td>Arbitrary labels related to the event</td>
    <td>Attributes[*]</td>
  </tr>
  <tr>
    <td>tags</td>
    <td>array of string</td>
    <td>List of values related to the event</td>
    <td>?</td>
  </tr>
  <tr>
    <td>trace.id</td>
    <td>string</td>
    <td>Trace ID</td>
    <td>TraceId</td>
  </tr>
  <tr>
    <td>span.id*</td>
    <td>string</td>
    <td>Span ID</td>
    <td>SpanId</td>
  </tr>
  <tr>
    <td>agent.ephemeral_id</td>
    <td>string</td>
    <td>Ephemeral ID created by agent</td>
    <td>**Resource</td>
  </tr>
  <tr>
    <td>agent.id</td>
    <td>string</td>
    <td>Unique identifier of this agent</td>
    <td>**Resource</td>
  </tr>
  <tr>
    <td>agent.name</td>
    <td>string</td>
    <td>Name given to the agent</td>
    <td>Resource["telemetry.sdk.name"]</td>
  </tr>
  <tr>
    <td>agent.type</td>
    <td>string</td>
    <td>Type of agent</td>
    <td>Resource["telemetry.sdk.language"]</td>
  </tr>
  <tr>
    <td>agent.version</td>
    <td>string</td>
    <td>Version of agent</td>
    <td>Resource["telemetry.sdk.version"]</td>
  </tr>
  <tr>
    <td>source.ip, client.ip</td>
    <td>string</td>
    <td>The IP address that the request was made from.</td>
    <td>Attributes["net.peer.ip"] or Attributes["net.host.ip"]</td>
  </tr>
  <tr>
    <td>cloud.account.id</td>
    <td>string</td>
    <td>ID of the account in the given cloud</td>
    <td>Resource["cloud.account.id"]</td>
  </tr>
  <tr>
    <td>cloud.availability_zone</td>
    <td>string</td>
    <td>Availability zone in which this host is running.</td>
    <td>Resource["cloud.zone"]</td>
  </tr>
  <tr>
    <td>cloud.instance.id</td>
    <td>string</td>
    <td>Instance ID of the host machine.</td>
    <td>**Resource</td>
  </tr>
  <tr>
    <td>cloud.instance.name</td>
    <td>string</td>
    <td>Instance name of the host machine.</td>
    <td>**Resource</td>
  </tr>
  <tr>
    <td>cloud.machine.type</td>
    <td>string</td>
    <td>Machine type of the host machine.</td>
    <td>**Resource</td>
  </tr>
  <tr>
    <td>cloud.provider</td>
    <td>string</td>
    <td>Name of the cloud provider. Example values are aws, azure, gcp, or digitalocean.</td>
    <td>Resource["cloud.provider"]</td>
  </tr>
  <tr>
    <td>cloud.region</td>
    <td>string</td>
    <td>Region in which this host is running.</td>
    <td>Resource["cloud.region"]</td>
  </tr>
  <tr>
    <td>cloud.image.id*</td>
    <td>string</td>
    <td></td>
    <td>Resource["host.image.name"]</td>
  </tr>
  <tr>
    <td>container.id</td>
    <td>string</td>
    <td>Unique container id</td>
    <td>Resource["container.id"]</td>
  </tr>
  <tr>
    <td>container.image.name</td>
    <td>string</td>
    <td>Name of the image the container was built on.</td>
    <td>Resource["container.image.name"]</td>
  </tr>
  <tr>
    <td>container.image.tag</td>
    <td>Array of string</td>
    <td>Container image tags.</td>
    <td>**Resource</td>
  </tr>
  <tr>
    <td>container.labels</td>
    <td>key/value</td>
    <td>Image labels.</td>
    <td>Attributes[*]</td>
  </tr>
  <tr>
    <td>container.name</td>
    <td>string</td>
    <td>Container name.</td>
    <td>Resource["container.name"]</td>
  </tr>
  <tr>
    <td>container.runtime</td>
    <td>string</td>
    <td>Runtime managing this container. Example: "docker"</td>
    <td>**Resource</td>
  </tr>
  <tr>
    <td>destination.address</td>
    <td>string</td>
    <td>Destination address for the event</td>
    <td>Attributes["destination.address"]</td>
  </tr>
  <tr>
    <td>error.code</td>
    <td>string</td>
    <td>Error code describing the error.</td>
    <td>Attributes["error.code"]</td>
  </tr>
  <tr>
    <td>error.id</td>
    <td>string</td>
    <td>Unique identifier for the error.</td>
    <td>Attributes["error.id"]</td>
  </tr>
  <tr>
    <td>error.message</td>
    <td>string</td>
    <td>Error message.</td>
    <td>Attributes["error.message"]</td>
  </tr>
  <tr>
    <td>error.stack_trace</td>
    <td>string</td>
    <td>The stack trace of this error in plain text.</td>
    <td>Attributes["error.stack_trace]</td>
  </tr>
  <tr>
    <td>host.architecture</td>
    <td>string</td>
    <td>Operating system architecture</td>
    <td>**Resource</td>
  </tr>
  <tr>
    <td>host.domain</td>
    <td>string</td>
    <td>Name of the domain of which the host is a member.

For example, on Windows this could be the host’s Active Directory domain or NetBIOS domain name. For Linux this could be the domain of the host’s LDAP provider.</td>

<td>**Resource</td>
  </tr>
  <tr>
    <td>host.hostname</td>
    <td>string</td>
    <td>Hostname of the host.

It normally contains what the hostname command returns on the host machine.</td>

<td>Resource["host.hostname"]</td>

  </tr>
  <tr>
    <td>host.id</td>
    <td>string</td>
    <td>Unique host id.</td>
    <td>Resource["host.id"]</td>
  </tr>
  <tr>
    <td>host.ip</td>
    <td>Array of string</td>
    <td>Host IP</td>
    <td>Resource["host.ip"]</td>
  </tr>
  <tr>
    <td>host.mac</td>
    <td>array of string</td>
    <td>MAC addresses of the host</td>
    <td>Resource["host.mac"]</td>
  </tr>
  <tr>
    <td>host.name</td>
    <td>string</td>
    <td>Name of the host.

It may contain what hostname returns on Unix systems, the fully qualified, or a name specified by the user. </td>

<td>Resource["host.name"]</td>

  </tr>
  <tr>
    <td>host.type</td>
    <td>string</td>
    <td>Type of host.</td>
    <td>Resource["host.type"]</td>
  </tr>
  <tr>
    <td>host.uptime</td>
    <td>string</td>
    <td>Seconds the host has been up.</td>
    <td>?</td>
  </tr>
  <tr>
    <td>service.ephemeral_id

</td>
    <td>string</td>
    <td>Ephemeral identifier of this service</td>
    <td>**Resource</td>
  </tr>
  <tr>
    <td>service.id</td>
    <td>string</td>
    <td>Unique identifier of the running service. If the service is comprised of many nodes, the service.id should be the same for all nodes.</td>
    <td>**Resource</td>
  </tr>
  <tr>
    <td>service.name</td>
    <td>string</td>
    <td>Name of the service data is collected from.</td>
    <td>Resource["service.name"]</td>
  </tr>
  <tr>
    <td>service.node.name</td>
    <td>string</td>
    <td>Specific node serving that service</td>
    <td>Resource["service.instance.id"]</td>
  </tr>
  <tr>
    <td>service.state</td>
    <td>string</td>
    <td>Current state of the service.</td>
    <td>Attributes["service.state"]</td>
  </tr>
  <tr>
    <td>service.type</td>
    <td>string</td>
    <td>The type of the service data is collected from.</td>
    <td>**Resource</td>
  </tr>
  <tr>
    <td>service.version</td>
    <td>string</td>
    <td>Version of the service the data was collected from.</td>
    <td>Resource["service.version"]</td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

\* Not yet formalized into ECS.

\*\* A resource that doesn’t exist in the
[OpenTelemetry resource semantic convention](../resource/semantic_conventions/README.md).

This is a selection of the most relevant fields. See
[for the full reference](https://www.elastic.co/guide/en/ecs/current/ecs-field-reference.html)
for an exhaustive list.

## References

