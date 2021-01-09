---
featured: true
title: "Applying Observability with OpenTelemetry - Part 1 - Distributed Tracing"
tags:
  [
	  golang,
	  cncf,
	  observability,
	  distributed-tracing,
	  opentelemetry,
	  jaeger,
	  prometheus,
	  distributed-context
  ]
date: 2021-01-03T14:24:10+10:00
lastmod: 2021-01-03T14:24:10+10:00
draft: false
---

<figure>
  <img src="/img/posts/2021-01-15-gotracing.png" alt="Gopher Tracing">
  <figcaption>By Lev Polyakov (www.polyakovproductions.com)</figcaption>
</figure>

Morden software systems nowadays are no longer running in a single server, but across multiple machines, databases, SaaS and cloud vendors. They are implemented in different programming languages and leverage a variety of communication protocols. As a result, measuring distributed systems' accuracy, latency, correctness and consistency is a thorny problem. Evolutions in architecture landscape especially with microservices, serverless, event-based even make the operation of the systems more challenging.

**Observability** (a new buzzword) is a quality of a software system that is implemented to generate enough data points to reason about the system during its operation. Fast feedback from production is crucial for an engineering team to improve and automate highly performant and reliable systems at scale. Three pillars of Observability covered in this series of blogs are **Distributed Tracing, Logging and Metrics**.

Previously, many existing OSS libraries and SaaS vendors are already presenting capabilities to export logs, metrics and traces out of software systems. Around May 2019, **OpenTelemetry (OTel)** was [announced](https://www.cncf.io/blog/2019/05/21/a-brief-history-of-opentelemetry-so-far/) as a CNCF sandbox project with a focus to standardise telemetry data model, architecture and implementation of how an observable software should be built. At the time of writing, implementations of OTel are in beta stage and GA release may be out soon in 2021 for major languages.

Since a few examples I found do not give me a comprehensive view of OTel as they're not solution-focused, I keen to assemble an example in Golang to play with some key features of OpenTelemetry. This custom solution is not an example of a production-ready one so please bear that in mind.

This blog post is the first part in a series about OpenTelemetry I plan to blog about:

- Part 1 - Distributed Tracing (this post)
- Part 2 - Logs and Metrics
- Part 3 - Cloud-native integration with AWS

{{% toc %}}

### Explore OTel with ShortenIt

ShortenIt solution provides similar functionalities as **tinyurl.com** website. Basically, we can generate a short URL from a very long one, short URLs save space for sharing, displaying or printing. In the following design, two main services `api-shortenit-v1` and `grpc-alias-provider-v1` are implemented with OTel to send its telemetry data (metrics, traces) to backend collectors (Prometheus and Jaeger).

![Shortenit Design](/img/posts/2021-01-15-shortenit.png 'Shortenit')

Two use cases we want to add distributed trace are:

##### Creating a short URL via POST /shortenit

A client sends a long URL to `api-shortenit-v1`, this service calls `grpc-alias-provider-v1` to get a short key and the URL mapping is persisted in MongoDB. If the request contains an authenticated user then we need to save the mapping to the user's URL list as well. We need obtain the request trace from the client to `api-shortenit-v1`, `grpc-alias-provider-v1`, MongoDB and Redis cache (alias store)

##### Get the original URL for an alias via GET /shortenit/{alias}

When the client wants to get back the original URL, `api-shortenit-v1` will check
the mapping in MongoDB, if found the long URL is returned for the provided key. For the analytical purpose, the API also publishes success responses to a Kafka topic.

If you keen to run and play with the example locally, please follow instructions outline <https://github.com/thanhnamit/shortenit>

### Distributed tracing in action

Now let's have a look at an end-to-end trace of the request `POST /shortenit`. Client app can extract initial `TraceId` and `SpanId` in response header `Traceparent`, for example `8d63ac3f01d517d71c5a803f3e37e981` is my unique TraceId. To view its trace timeline, enter the Id in Jeager at `http://localhost:16686/search`.

![Trace Timeline](/img/posts/2021-01-15-tracetimeline.png 'Trace Timeline')

What the trace timeline telling us:

- Number of participants: 2
- The latency of request: 7.33ms
- Number of operations (spans) and execution time for each operation: 17
- Causality relationships between spans

This single end-to-end view of a request is extremely valuable for engineers to analyse and detect abnormality if service quality degraded. Additionally, each span also carries information collected at the time the operation executed, for example:

![Span Details](/img/posts/2021-01-15-spandetails.png 'Span Details')

This span is instrumented by otelgrpc auto-instrumentation library, which is prebuilt for a specific protocol (grpc in this case). It collects information about rpc status code, service name & method, and host information. Span can be customised in application code to capture operation details if more visibility of the trace is required.

Let's look at Distributed tracing from the implementation perspective. The language I used here is Golang but a similar approach should be available for other languages as they all conform to the same OTel specification.

#### Configuring Exporter, Tracer, Sampler and Propagator

The first step is to config necessary component for tracing. OTel Tracing API standardises the following concepts:

- **Exporter**: a protocol-specific, pluggable component that implements OTel's SpanExporter interface to send telemetry data to backend (i.e Jaeger in this example)
- **TraceProvider**: responsible for creating a Tracer
- **Tracer**: is responsible for making `Span`
- **Sampler**: controls when a span is recorded
- **Propagator**: responsible to read and write context data across process boundaries

The following code configs a full pipeline backed by Jaeger's collector endpoint normally available at <http://127.0.0.1:14268/api/traces>. TraceProvider, TextMapPropagator and AlwaysSample sampler are configured and registered as global variables.

```go
func InitTracer(serviceName string, traceCollector string) func() {
	flush, err := jaeger.InstallNewPipeline(
		jaeger.WithCollectorEndpoint(traceCollector),
		...
		jaeger.WithSDK(&sdktrace.Config{DefaultSampler: sdktrace.AlwaysSample()}),
	)
	...
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(propagation.TraceContext{}, propagation.Baggage{}))
	return flush
}
```

With that configuration, let's look at several instrumentation use cases:

#### Custom span and in-process context passing

Tracing an individual function with blocks of logic could give us a meaningful measurement of how the main logic performs. An application should be able
to propagate in-process context in its call stack so an existing span can be extracted for linking and a new span to be inserted. Depend on the language we choose, Context is available explicitly (i.e Golang) or implicitly via Thread-local vars (i.e Java). In this instance, I created a custom span called `service.GetNewAlias` that comprises multiple child spans when the main service function calls other functions to fulfil its logic.

![Custom Span](/img/posts/2021-01-15-customspan.png 'Custom Span')

To create a new span in Go, we obtain an instance of the tracer. If the function has access to an existing context, it can pass the context to OTel `trace.Start` function to create a new child span.

```go 
// main function
func (d DefaultService) GetNewAlias(ctx context.Context, request core.ShortenURLRequest) (core.ShortenURLResponse, error) {
	tr := otel.Tracer(d.cfg.TracerName)
	ctx, span := tr.Start(ctx, "service.GetNewAlias")
	defer span.End()
	...

// child function
func (r *UserRepo) GetUserByEmail(ctx context.Context, email string) (*core.User, error) {
	tr := otel.Tracer(r.cfg.TracerName)
	ctx, span := tr.Start(ctx, "repository.user.GetUserByEmail")
	defer span.End()
```

In-process context passing is not new as it has been implemented in many frameworks. Inter-process context passing, however, requires a different strategy for interoperability.

#### Tracing remote calls and propagating inter-process context

In a lot of situations, our service will need to call an upstream service, hence this is an ideal point for instrumentation. In Shortenit case, `api-shortenit-v1` calls `grpc-alias-provider-v1` using Grpc protocol. As you might guess, there will be two spans created, one on the caller side and one on the receiver side.

![Cross boundaries](/img/posts/2021-01-15-crossprocess.png 'Cross boundaries')

As above, all we need is to add dependencies and configure on client and server

```go
go get -u go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc@v0.15.0
```

Grpc client needs to be configured with an Interceptor

```go
func NewAliasClient(cfg *config.Config) *AliasClient {
	conn, err := grpc.DialContext(context.Background(), cfg.AliasCon, grpc.WithInsecure(), grpc.WithBlock(), grpc.WithUnaryInterceptor(otelgrpc.UnaryClientInterceptor()))
    //...
}
```

An inceptor on server side

```go
s := grpc.NewServer(
    grpc.UnaryInterceptor(otelgrpc.UnaryServerInterceptor()),
)
```

It sounds like magic but if we look at the Interceptor code all it does is interacting with OTel SDK's propagators to inject and extract data from an outgoing and an incoming request. Currently, OTel SDKs bundled with W3C Trace Context implementation as default.

#### Instrumenting HTTP listener

Instrumenting at listener level provides valuable information about how a client interacts with a REST service; we can record http related attributes in Span tags (method, path, user_agent, status code...).

![Http Listener](/img/posts/2021-01-15-httplistener.png 'Http Listener')

The instrumentation library is `otelhttp`, not part of OTel spec but provided by community or vendor. We need to add this dependency and register it as middleware in http pipeline.

```go
go get -u go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp@v0.15.0
```

```go
func NewGlobalHandler(handler http.Handler, operation string) func(w http.ResponseWriter, r *http.Request) {
	return toHandlerFunc(withCORS(withAPIKey(otelhttp.NewHandler(handler, operation))))
}
```

#### Instrumenting database and backends

As a lot of performance issues are database-related, instrumenting database interaction hence is a crucial part to analyse and troubleshoot the issues. Commercial and OSS database vendors start adopting OTel instrumentation by providing their version of library. The span below, created by `otelmongo`, expose details about the query execution and database instance that our API interacts with.

![Mongo](/img/posts/2021-01-15-mongo.png 'Mongo')

```go 
go get -u go.opentelemetry.io/contrib/instrumentation/go.mongodb.org/mongo-driver/mongo/otelmongo
```

Setting up new mongo connection with additional monitor

```go
opts := options.Client()
opts.Monitor = otelmongo.NewMonitor(cfg.AppName)
opts.ApplyURI(cfg.MongoCon)
db, err := mongo.NewClient(opts)
```

#### Instrumentation for messaging

Instrumenting for messaging or asynchronous calls is not always easy as it involves a message queue in between and context has to be passed together with messages ( requires a change in message structures). Shopify's engineering team has contributed `otelsarama` library to support Kafka client/server instrumentation.

![Kafka](/img/posts/2021-01-15-kafka.png 'Kafka')

```go
go get -u "go.opentelemetry.io/contrib/instrumentation/github.com/Shopify/sarama/otelsarama"
```

Configuration to Sarama's producer:

```go
...
producer, err := sarama.NewAsyncProducer(brokerList, cfg)
...
producer = otelsarama.WrapAsyncProducer(cfg, producer)
...
```

With that we need to explicitly inject the metadata using the configured propagator before sending a message to a topic:

```go
...
otel.GetTextMapPropagator().Inject(ctx, otelsarama.NewProducerMessageCarrier(&msg))
ep.producer.Input() <- &msg
...
```

#### Web and mobile instrumentation

Most of frontend today interact with REST api via HTTP, so this scenario is identical to remote call instrumentation. However, since frontend sits independently from backend, a trust layer need to be established to make sure tracing data is
valid and correctly adjusted before assembling traces.

### Summary

This is the first pillar of Observability, its main goal is to enable engineers to understand and debug programmatic failures in a distributed environment. In the next post, we will look at how OTel handles logs and metrics. Stay tuned!

### References

- [Master Distributed Tracing by Yuri Shkuro](https://www.packtpub.com/product/mastering-distributed-tracing/9781788628464) - Highly recommended
- [OpenTelemetry specification](https://github.com/open-telemetry/opentelemetry-specification)
- [OTel Go](https://github.com/open-telemetry/opentelemetry-go)
- [OTel Go Contrib](https://github.com/open-telemetry/opentelemetry-go-contrib)