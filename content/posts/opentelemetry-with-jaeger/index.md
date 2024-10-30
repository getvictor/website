+++
title = "Is OpenTelemetry userful for the average software developer?"
description = "First impressions of OpenTelemetry with Jaeger and use cases for software developers"
authors = ["Victor Lyuboslavsky"]
image = "opentelemetry-with-jaeger-headline.png"
date = 2024-10-30
categories = ["Software Development", "DevOps & Infrastructure"]
tags = ["Telemetry", "DevTools", "Performance", "Golang"]
draft = false
+++

This article discusses our first impressions of using OpenTelemetry with Jaeger.

- [Use cases for OpenTelemetry and Jaeger](#use-cases-for-opentelemetry-and-jaeger)
- [Problems with OpenTelemetry and Jaeger](#problems-with-opentelemetry-and-jaeger)

## What is OpenTelemetry?

[OpenTelemetry](https://opentelemetry.io/) is a set of APIs, libraries, agents, and instrumentation for collecting
distributed traces and metrics from your applications. It provides a standardized way to instrument your code and
collect telemetry data. OpenTelemetry supports programming languages like Java, Python, Go, JavaScript, etc.

Tracing is a method of monitoring and profiling your application to understand how requests flow through your system.
For example, you can view the associated database calls and requests to other services for a single API request. Tracing
allows you to identify bottlenecks, latency issues, and other performance problems.

## What is Jaeger?

[Jaeger](https://www.jaegertracing.io/) is an open-source, end-to-end distributed tracing system. Jaeger is popular for
tracing applications because of its scalability, ease of use, and integration with other tools. Jaeger provides a
web-based UI for viewing traces and analyzing performance data.

## Add OpenTelemetry instrumentation to your application

To start with OpenTelemetry and Jaeger, you must instrument your application with OpenTelemetry libraries.

In our case, we used the OpenTelemetry Go SDK to instrument our Go application. We added the necessary dependencies to
our project.

```
go get go.opentelemetry.io/otel@v1.31.0
go get go.opentelemetry.io/otel/exporters/otlp/otlptrace@v1.31.0
go get go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc@v1.31.0
go get go.opentelemetry.io/otel/sdk@v1.31.0
go get go.opentelemetry.io/contrib/instrumentation/github.com/gorilla/mux/otelmux@v0.56.0
go get github.com/XSAM/otelsql@v0.35.0
```

The `go.opentelemetry.io/contrib/instrumentation/github.com/gorilla/mux/otelmux` package is needed to instrument our
`gorilla/mux` HTTP router.

```go
    r := mux.NewRouter()
    r.Use(otelmux.Middleware("fleet"))
```

The `github.com/XSAM/otelsql` package is needed to instrument our SQL database queries.

```go
// ...
import "github.com/XSAM/otelsql"
import semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
// ...
var otelTracedDriverName string
func init() {
    var err error
    otelTracedDriverName, err = otelsql.Register("mysql",
       otelsql.WithAttributes(semconv.DBSystemMySQL),
       otelsql.WithSpanOptions(otelsql.SpanOptions{
          // DisableErrSkip ignores driver.ErrSkip errors, which are frequently returned by the MySQL
          // driver when certain optional methods or paths are not implemented/taken.
          // For example, interpolateParams=false (the secure default) will not do a parametrized
          // sql.conn.query directly without preparing it first, causing driver.ErrSkip
          DisableErrSkip: true,
          // Omitting span for sql.conn.reset_session since it takes ~1us and doesn't provide useful
          // information
          OmitConnResetSession: true,
          // Omitting span for sql.rows since it is very quick and typically doesn't provide useful
          // information beyond what's already reported by prepare/exec/query
          OmitRows: true,
       }),
       // WithSpanNameFormatter allows us to customize the span name, which is especially useful for SQL
       // queries run outside an HTTPS transaction, which do not belong to a parent span, show up as their
       // own trace, and would otherwise be named "sql.conn.query" or "sql.conn.exec".
       otelsql.WithSpanNameFormatter(func(ctx context.Context, method otelsql.Method, query string) string {
          if query == "" {
             return string(method)
          }
          // Append query with extra whitespaces removed
          query = strings.Join(strings.Fields(query), " ")
          if len(query) > 100 {
             query = query[:100] + "..."
          }
          return string(method) + ": " + query
       }),
    )
    if err != nil {
       panic(err)
    }
}
```

Then, use `otelTracedDriverName` to open a connection to your database.

```go
    db, err := sql.Open(otelTracedDriverName, "user:password@tcp(localhost:3306)/database")
```

When starting your application, you must create an OpenTelemetry exporter and a trace provider.

```go
    ctx := context.Background()
    client := otlptracegrpc.NewClient()
    otlpTraceExporter, err := otlptrace.New(ctx, client)
    if err != nil {
        panic("Failed to initialize tracing")
    }
    batchSpanProcessor := trace.NewBatchSpanProcessor(otlpTraceExporter)
    tracerProvider := trace.NewTracerProvider(trace.WithSpanProcessor(batchSpanProcessor))
    otel.SetTracerProvider(tracerProvider)
```

## Launch Jaeger

To view traces, you need to launch Jaeger. You can run Jaeger locally using Docker. Based on the
[Jaeger 1.62 Getting Started guide](https://www.jaegertracing.io/docs/1.62/getting-started/), you can run the following
command:

```bash
docker run --rm --name jaeger \
  -p 16686:16686 \
  -p 4317:4317 \
jaegertracing/all-in-one:1.62.0
```

In our example, we are only exposing two ports:

- `4317` for the Jaeger collector, which receives trace data using OpenTelemetry Protocol (OTLP) over gRPC
- `16686` for the Jaeger UI

## Launch your application

Before starting your application, you must set the OpenTelemetry endpoint to send traces to Jaeger. For example:

```bash
export OTEL_SERVICE_NAME=fleet
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

Now, you can start your application.

## View traces in Jaeger

Open your browser and navigate to [http://localhost:16686](http://localhost:16686) to view traces in the Jaeger UI.
Select your **Service** name and click **Find Traces**.

You can click into a trace to view the details of each span. You can see the duration, logs, and tags for each span. The
example below shows the HTTP request details and multiple SQL queries.

{{< figure src="example-jaeger-trace.png" alt="Fleet hosts request with SQL queries to sessions, users, user_teams, and host tables.">}}

## Use cases for OpenTelemetry and Jaeger

In a local software development environment, OpenTelemetry and Jaeger can be used to:

- Fix bottlenecks and latency issues
- Understand how requests flow through your system

If a bottleneck is known or suspected, Jaeger can help you identify the root cause. For example, you can see which
database queries are taking the most time and optimize them.

When developing new features, Jaeger can help you understand how requests flow through your system. This telemetry data
provides a quick check to ensure your new feature works as expected.

In a production environment, OpenTelemetry and Jaeger can be used to:

- Monitor and profile your applications
- Troubleshoot performance issues
- Optimize your applications and improve user experience
- Ensure your applications meet service level objectives (SLOs)

## Problems with OpenTelemetry and Jaeger

OpenTelemetry and Jaeger are powerful tools, yet their development use seems limited to fixing performance bottlenecks.
They cannot be used for general debugging out of the box since they don't provide enough detail for each specific
request, such as the request body.

In addition, missing spans can be a problem. If your application is not instrumented correctly, you may not see all the
spans you expect or know about in Jaeger. Our application lacks spans for some API endpoints, Redis transactions,
outbound HTTP requests, and asynchronous processes. Adding all of these spans requires additional development and QA
efforts.

The Jaeger UI itself is basic and lacks some features. For example, regex search is missing out of the box, unless
Elasticsearch/OpenSearch storage is added.

Our chosen SQL instrumentation library, [github.com/XSAM/otelsql](https://github.com/XSAM/otelsql), could be better. It
does not provide a way to trace the transaction lifecycle, and it creates many spans at the root level, clogging the
Jaeger UI.

## Further reading

- Recently, we wrote about [benchmarking the performance of your Go code](../optimizing-performance-of-go-app)

## Example code on GitHub

[Fleet Device Management repo with OpenTelemetry instrumentation (as of this writing)](https://github.com/fleetdm/fleet/tree/6bc0b5dcd9214c6e3ff94fe657947aeccbdad352)

## Watch OpenTelemetry with Jaeger video

{{< youtube eQhdvU2gsmQ >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
