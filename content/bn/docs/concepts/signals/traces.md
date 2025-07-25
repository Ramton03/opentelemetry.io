---
title: Traces
weight: 1
description: অ্যাপ্লিকেশনের মধ্য দিয়ে একটি অনুরোধের সম্পূর্ণ "path"।
cSpell:ignore: Guten
---


**Traces** আমাদের যখন কোনো অনুরোধ অ্যাপ্লিকেশনে পাঠানো হয় তখন কী ঘটে তার সার্বিক চিত্র দেখায়। আপনার অ্যাপ্লিকেশন যদি একক ডাটাবেসের monolith অথবা বিভিন্ন সার্ভিসের সমন্বয়ে গঠিত service mesh হয়, traces ছাড়া অনুরোধটি অ্যাপ্লিকেশনে যে সম্পূর্ণ "path" অনুসরণ করে তা বোঝা সম্ভব নয়।

চলুন তিনটি কাজের ইউনিটের মাধ্যমে এটি অন্বেষণ করি, যেগুলো নিম্নরূপ উপস্থাপিত [Spans](#spans):

{{% alert title="Note" %}}

নিম্নলিখিত JSON উদাহরণগুলো কোনো নির্দিষ্ট ফরম্যাট উপস্থাপন করে না, বিশেষ করে এগুলো [OTLP/JSON](/docs/specs/otlp/#json-protobuf-encoding)নয়, যা আরও verbose।

{{% /alert %}}

`hello` span:

```json
{
  "name": "hello",
  "context": {
    "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
    "span_id": "051581bf3cb55c13"
  },
  "parent_id": null,
  "start_time": "2022-04-29T18:52:58.114201Z",
  "end_time": "2022-04-29T18:52:58.114687Z",
  "attributes": {
    "http.route": "some_route1"
  },
  "events": [
    {
      "name": "Guten Tag!",
      "timestamp": "2022-04-29T18:52:58.114561Z",
      "attributes": {
        "event_attributes": 1
      }
    }
  ]
}
```

এটি হলো root span, যা সমগ্র অপারেশনটির শুরু এবং শেষ নির্দেশ করে। লক্ষ্য করুন এতে একটি `trace_id` ফিল্ড রয়েছে trace নির্দেশ করার জন্য, কিন্তু `parent_id` ফিল্ড নেই। এভাবেই আপনি জানতে পারবেন এটি root span।

`hello-greetings` span:

```json
{
  "name": "hello-greetings",
  "context": {
    "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
    "span_id": "5fb397be34d26b51"
  },
  "parent_id": "051581bf3cb55c13",
  "start_time": "2022-04-29T18:52:58.114304Z",
  "end_time": "2022-04-29T22:52:58.114561Z",
  "attributes": {
    "http.route": "some_route2"
  },
  "events": [
    {
      "name": "hey there!",
      "timestamp": "2022-04-29T18:52:58.114561Z",
      "attributes": {
        "event_attributes": 1
      }
    },
    {
      "name": "bye now!",
      "timestamp": "2022-04-29T18:52:58.114585Z",
      "attributes": {
        "event_attributes": 1
      }
    }
  ]
}
```

এই span নির্দিষ্ট কাজগুলোকে অন্তর্ভুক্ত করে, যেমন শুভেচ্ছা জানানো, এবং এর parent হলো `hello` span। লক্ষ্য করুন এটি একই `trace_id` শেয়ার করে root span-এর সাথে, যা নির্দেশ করে এটি একই trace-এর অংশ। এছাড়াও, এর একটি `parent_id` রয়েছে যা `hello` span-এর `span_id` এর সাথে মেলে।

`hello-salutations` span:

```json
{
  "name": "hello-salutations",
  "context": {
    "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
    "span_id": "93564f51e1abe1c2"
  },
  "parent_id": "051581bf3cb55c13",
  "start_time": "2022-04-29T18:52:58.114492Z",
  "end_time": "2022-04-29T18:52:58.114631Z",
  "attributes": {
    "http.route": "some_route3"
  },
  "events": [
    {
      "name": "hey there!",
      "timestamp": "2022-04-29T18:52:58.114561Z",
      "attributes": {
        "event_attributes": 1
      }
    }
  ]
}
```

এই spanটি এই trace-এর তৃতীয় operation প্রদর্শন করে এবং আগেরটির মতোই এটি `hello` span-এর child। এর ফলে এটি `hello-greetings` span-এর sibling-ও হয়।

এই তিনটি JSON ব্লকই একই `trace_id` শেয়ার করে, আর `parent_id` ফিল্ড একটি hierarchy নির্দেশ করে। তাই এটাই একটি Trace!

আরেকটি বিষয় আপনি লক্ষ্য করবেন, প্রতিটি Span structured log-এর মতোই দেখায়। কারণ এটাই! একভাবে ভাবতে পারেন, `Traces` হচ্ছে context, correlation, hierarchy ইত্যাদি অন্তর্নিহিত থাকা structured logs-এর একটি collection। তবে এই “structured logs” বিভিন্ন প্রক্রিয়াসমূহ, সেবাসমূহ, ভার্চুয়াল মেশিনসমূহ, ডেটা কেন্দ্রসমূহ ইত্যাদি থেকে আসতে পারে। এ কারণেই `tracing` যেকোনো সিস্টেমের সম্পূর্ণ প্রক্রিয়ার দৃশ্য উপস্থাপন করতে সক্ষম।

OpenTelemetry-এ tracing কীভাবে কাজ করে তা বুঝতে, চলুন আমাদের কোড ইন্সট্রুমেন্টেশনে অংশগ্রহণকারী উপাদানগুলোর তালিকা দেখি।

## Tracer সরবরাহকারী

একটি Tracer সরবরাহকারী (যাকে কখনো কখনো `TracerProvider` বলা হয়) হলো একটি factory যা `Tracer` তৈরি করে। অধিকাংশ অ্যাপ্লিকেশনে, একটি Tracer Provider একবার শুরু করা হয় এবং এর জীবনচক্র অ্যাপ্লিকেশনের জীবনচক্র এর সাথে মেলে। Tracer Provider initialization-এ সম্পদ এবং Exporter initialization ও অন্তর্ভুক্ত থাকে। এটি সাধারণত OpenTelemetry-তে tracing শুরু করার প্রথম ধাপ। কিছু language SDKs-এ একটি সার্বজনীন Tracer Provider ইতিমধ্যেই আপনার জন্য শুরু করা থাকে।

## Tracer

একটি Tracer স্প্যান তৈরি করে যা একটি নির্দিষ্ট operation-এ কী ঘটছে তার আরও তথ্য ধারণ করে, যেমন সার্ভিসে একটি অনুরোধের ক্ষেত্রে। Tracer-গুলো Tracer সরবরাহকারী থেকে তৈরি হয়।

## Trace এক্সপোর্টারস

Trace এক্সপোর্টারস traces-কে গ্রাহক-এর কাছে পাঠায়। এই গ্রাহক হতে পারে স্ট্যান্ডার্ড আউটপুট (ডিবাগিং এবং উন্নয়ন সময়ে), OpenTelemetry Collector, অথবা আপনার পছন্দমতো কোনো ওপেন সোর্স বা ভেন্ডর ব্যাকএন্ড।

## Context Propagation

Context Propagation is the core concept that enables Distributed Tracing. With
Context Propagation, Spans can be correlated with each other and assembled into
a trace, regardless of where Spans are generated. To learn more about this
topic, see the concept page on [Context Propagation](../../context-propagation).

## Spans

A **span** represents a unit of work or operation. Spans are the building blocks
of Traces. In OpenTelemetry, they include the following information:

- Name
- Parent span ID (empty for root spans)
- Start and End Timestamps
- [Span Context](#span-context)
- [Attributes](#attributes)
- [Span Events](#span-events)
- [Span Links](#span-links)
- [Span Status](#span-status)

Sample span:

```json
{
  "name": "/v1/sys/health",
  "context": {
    "trace_id": "7bba9f33312b3dbb8b2c2c62bb7abe2d",
    "span_id": "086e83747d0e381e"
  },
  "parent_id": "",
  "start_time": "2021-10-22 16:04:01.209458162 +0000 UTC",
  "end_time": "2021-10-22 16:04:01.209514132 +0000 UTC",
  "status_code": "STATUS_CODE_OK",
  "status_message": "",
  "attributes": {
    "net.transport": "IP.TCP",
    "net.peer.ip": "172.17.0.1",
    "net.peer.port": "51820",
    "net.host.ip": "10.177.2.152",
    "net.host.port": "26040",
    "http.method": "GET",
    "http.target": "/v1/sys/health",
    "http.server_name": "mortar-gateway",
    "http.route": "/v1/sys/health",
    "http.user_agent": "Consul Health Check",
    "http.scheme": "http",
    "http.host": "10.177.2.152:26040",
    "http.flavor": "1.1"
  },
  "events": [
    {
      "name": "",
      "message": "OK",
      "timestamp": "2021-10-22 16:04:01.209512872 +0000 UTC"
    }
  ]
}
```

Spans can be nested, as is implied by the presence of a parent span ID: child
spans represent sub-operations. This allows spans to more accurately capture the
work done in an application.

### Span Context

Span context is an immutable object on every span that contains the following:

- The Trace ID representing the trace that the span is a part of
- The span's Span ID
- Trace Flags, a binary encoding containing information about the trace
- Trace State, a list of key-value pairs that can carry vendor-specific trace
  information

Span context is the part of a span that is serialized and propagated alongside
[Distributed Context](#context-propagation) and [Baggage](../baggage).

Because Span Context contains the Trace ID, it is used when creating
[Span Links](#span-links).

### Attributes

Attributes are key-value pairs that contain metadata that you can use to
annotate a Span to carry information about the operation it is tracking.

For example, if a span tracks an operation that adds an item to a user's
shopping cart in an eCommerce system, you can capture the user's ID, the ID of
the item to add to the cart, and the cart ID.

You can add attributes to spans during or after span creation. Prefer adding
attributes at span creation to make the attributes available to SDK sampling. If
you have to add a value after span creation, update the span with the value.

Attributes have the following rules that each language SDK implements:

- Keys must be non-null string values
- Values must be a non-null string, boolean, floating point value, integer, or
  an array of these values

Additionally, there are
[Semantic Attributes](/docs/specs/semconv/general/trace/), which are known
naming conventions for metadata that is typically present in common operations.
It's helpful to use semantic attribute naming wherever possible so that common
kinds of metadata are standardized across systems.

### Span Events

A Span Event can be thought of as a structured log message (or annotation) on a
Span, typically used to denote a meaningful, singular point in time during the
Span's duration.

For example, consider two scenarios in a web browser:

1. Tracking a page load
2. Denoting when a page becomes interactive

A Span is best used to the first scenario because it's an operation with a start
and an end.

A Span Event is best used to track the second scenario because it represents a
meaningful, singular point in time.

#### When to use span events versus span attributes

Since span events also contain attributes, the question of when to use events
instead of attributes might not always have an obvious answer. To inform your
decision, consider whether a specific timestamp is meaningful.

For example, when you're tracking an operation with a span and the operation
completes, you might want to add data from the operation to your telemetry.

- If the timestamp in which the operation completes is meaningful or relevant,
  attach the data to a span event.
- If the timestamp isn't meaningful, attach the data as span attributes.

### Span Links

Links exist so that you can associate one span with one or more spans, implying
a causal relationship. For example, let’s say we have a distributed system where
some operations are tracked by a trace.

In response to some of these operations, an additional operation is queued to be
executed, but its execution is asynchronous. We can track this subsequent
operation with a trace as well.

We would like to associate the trace for the subsequent operations with the
first trace, but we cannot predict when the subsequent operations will start. We
need to associate these two traces, so we will use a span link.

You can link the last span from the first trace to the first span in the second
trace. Now, they are causally associated with one another.

Links are optional but serve as a good way to associate trace spans with one
another.

For more information see [Span Links](/docs/specs/otel/trace/api/#link).

### Span Status

Each span has a status. The three possible values are:

- `Unset`
- `Error`
- `Ok`

The default value is `Unset`. A span status that is `Unset` means that the
operation it tracked successfully completed without an error.

When a span status is `Error`, then that means some error occurred in the
operation it tracks. For example, this could be due to an HTTP 500 error on a
server handling a request.

When a span status is `Ok`, then that means the span was explicitly marked as
error-free by the developer of an application. Although this is unintuitive,
it's not required to set a span status as `Ok` when a span is known to have
completed without error, as this is covered by `Unset`. What `Ok` does is
represent an unambiguous "final call" on the status of a span that has been
explicitly set by a user. This is helpful in any situation where a developer
wishes for there to be no other interpretation of a span other than
"successful".

To reiterate: `Unset` represents a span that completed without an error. `Ok`
represents when a developer explicitly marks a span as successful. In most
cases, it is not necessary to explicitly mark a span as `Ok`.

### Span Kind

When a span is created, it is one of `Client`, `Server`, `Internal`, `Producer`,
or `Consumer`. This span kind provides a hint to the tracing backend as to how
the trace should be assembled. According to the OpenTelemetry specification, the
parent of a server span is often a remote client span, and the child of a client
span is usually a server span. Similarly, the parent of a consumer span is
always a producer and the child of a producer span is always a consumer. If not
provided, the span kind is assumed to be internal.

For more information regarding SpanKind, see
[SpanKind](/docs/specs/otel/trace/api/#spankind).

#### Client

A client span represents a synchronous outgoing remote call such as an outgoing
HTTP request or database call. Note that in this context, "synchronous" does not
refer to `async/await`, but to the fact that it is not queued for later
processing.

#### Server

A server span represents a synchronous incoming remote call such as an incoming
HTTP request or remote procedure call.

#### Internal

Internal spans represent operations which do not cross a process boundary.
Things like instrumenting a function call or an Express middleware may use
internal spans.

#### Producer

Producer spans represent the creation of a job which may be asynchronously
processed later. It may be a remote job such as one inserted into a job queue or
a local job handled by an event listener.

#### Consumer

Consumer spans represent the processing of a job created by a producer and may
start long after the producer span has already ended.

## Specification

For more information, see the
[traces specification](/docs/specs/otel/overview/#tracing-signal).