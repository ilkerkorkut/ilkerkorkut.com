---
title: "All-in-One Solution for observability in Go Microservices using Prometheus, Loki, Promtail, Grafana and Tempo"
author: ilker
date: 2024-06-26
tags:
  - prometheus
  - loki
  - promtail
  - grafana
  - tempo
  - go
  - golang
categories:
  - go
  - microservices
  - observability
  - monitoring
  - distributed tracing
keywords:
  - prometheus
  - loki
  - promtail
  - grafana
  - tempo
  - go
  - tracing
  - distributed
---


<img src="/images/go-observability/gopher-sailor.jpeg" alt="Go Microservice Observability" style="width:auto;height:300px;">

In this post, I will try to demonstrate how to build a complete observability solution for a go microservice with Prometheus, Loki, Promtail, Tempo and interacting with the Grafana dashboard. In API layer, I will use Fiber framework for building a simple service. If you are in a hurry, you can find the source code in [this](#source-code-and-how-to-run-the-application) section.

Prerequisites:

- Go 1.22
- Docker (and docker-compose)

While gathering logs, I also create a kafka sink to consuming logs on promtail for loki. Promtail has flexibility to gather logs from files or different data sources as well.

## Architecture

![Go Observability Architecture](/images/go-observability/go-observability.png)

## Prometheus Integration

We create a factory for Prometheus integration so that, this factory can handle the registring the prometheus registry and also http handler for prometheus metrics which can be used in the application.

```go
type PrometheusFactory interface {
	InitHandler() http.Handler
}

type prometheusFactory struct {
	registry *prometheus.Registry
}

func NewPrometheusFactory(cs ...prometheus.Collector) PrometheusFactory {
	f := &prometheusFactory{}
	f.registry = f.initRegistry(cs)
	return f
}

func (f *prometheusFactory) InitHandler() http.Handler {
	return promhttp.HandlerFor(f.registry, promhttp.HandlerOpts{Registry: f.registry})
}

func (f *prometheusFactory) initRegistry(cs []prometheus.Collector) *prometheus.Registry {
	reg := prometheus.NewRegistry()
	defaultCs := []prometheus.Collector{
		collectors.NewGoCollector(),
		collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}),
	}

	fCS := make([]prometheus.Collector, 0, len(defaultCs)+len(cs))
	fCS = append(fCS, defaultCs...)

	for _, c := range cs {
		if c != nil {
			fCS = append(fCS, c)
		}
	}

	reg.MustRegister(fCS...)

	return reg
}
```

When we started the application with the following factory instance and tying the handler to the `/metrics` endpoint, we can see the metrics on the response.

Following code blocks, shows us how to bind the handler to the go-fiber router. Actually, it is basically *net/http* handler.

```go
app.Get("/metrics", adaptor.HTTPHandler(a.PrometheusFactory.InitHandler()))
```

During this definition, just to be make sure it shouldn't be under any business logic related middleware or that can affect the metrics. By applying this approach, we can play with any metrics on any middleware or handler.

## Sending Custom Metrics to Prometheus

We can define any custom metrics metrics and register them to the prometheus registry. For instance, on following code block we defined a counter metric for counting the requests.

```go
package metric

import "github.com/prometheus/client_golang/prometheus"

var TestCounter = prometheus.NewCounter(prometheus.CounterOpts{
	Namespace:   "myapp",
	Subsystem:   "api",
	Name:        "request_counter",
	Help:        "Counter for requests",
	ConstLabels: nil,
})
```

Registering the custom metrics to the prometheus registry is also simple as following:

```go
promFactory := factory.NewPrometheusFactory(
		metric.TestCounter,
)
```

Also we can create a request counter to observe that custom metric inside the application.

```go
package middleware

import (
	"github.com/gofiber/fiber/v2"
	"github.com/ilkerkorkut/go-examples/microservice/observability/internal/metric"
)

func MetricsMiddleware() func(c *fiber.Ctx) error {
	return func(c *fiber.Ctx) (err error) {
		metric.TestCounter.Add(1)
		return c.Next()
	}
}
```

## Setting up Logging with Zap to Sink into Kafka

Zap logging library is commonly used in go applications. It is fast and flexible. So, we will use it in our application. We can define a logger factory to create a logger instance with some configurations.


First we can create a kafka sink for the logger. We will use IBM/sarama kafka client for that purpose.

```go
package logging

import (
	"time"

	"github.com/IBM/sarama"
)

type kafkaSink struct {
	producer sarama.SyncProducer
	topic    string
}

func (s kafkaSink) Write(b []byte) (int, error) {
	_, _, err := s.producer.SendMessage(&sarama.ProducerMessage{
		Topic: s.topic,
		Key:   sarama.StringEncoder(time.Now().String()),
		Value: sarama.ByteEncoder(b),
	})
	return len(b), err
}

func (s kafkaSink) Sync() error {
	return nil
}

func (s kafkaSink) Close() error {
	return nil
}
```

You can ask why there is `Sync` and `Close` methods.

When we take a look tha zap's `Sink` interface, it derives from `WriteSyncer` and `io.Closer` interfaces. Because of that we need to implement all of the methods, this is golang interface's nature.

```go
type Sink interface {
	zapcore.WriteSyncer
	io.Closer
}
```
Then, we can create a simple application logger initializer as follows;

```go
package logging

import (
	"fmt"
	"log"
	"net/url"
	"strings"

	"github.com/IBM/sarama"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

func InitLogger(lvl string, kafkaBrokers []string) (*zap.Logger, error) {
	level, err := parseLevel(lvl)
	if err != nil {
		return nil, err
	}

	return getLogger(level, kafkaBrokers), nil
}

func getLogger(
	level zapcore.Level,
	kafkaBrokers []string,
) *zap.Logger {
	kafkaURL := url.URL{Scheme: "kafka", Host: strings.Join(kafkaBrokers, ",")}

	ec := zap.NewProductionEncoderConfig()
	ec.EncodeName = zapcore.FullNameEncoder
	ec.EncodeName = zapcore.FullNameEncoder
	ec.EncodeTime = zapcore.RFC3339TimeEncoder
	ec.EncodeDuration = zapcore.MillisDurationEncoder
	ec.EncodeLevel = zapcore.CapitalLevelEncoder
	ec.EncodeCaller = zapcore.ShortCallerEncoder
	ec.NameKey = "logger"
	ec.MessageKey = "message"
	ec.LevelKey = "level"
	ec.TimeKey = "timestamp"
	ec.CallerKey = "caller"
	ec.StacktraceKey = "trace"
	ec.LineEnding = zapcore.DefaultLineEnding
	ec.ConsoleSeparator = " "

	zapConfig := zap.Config{}
	zapConfig.Level = zap.NewAtomicLevelAt(level)
	zapConfig.Encoding = "json"
	zapConfig.EncoderConfig = ec
	zapConfig.OutputPaths = []string{"stdout", kafkaURL.String()}
	zapConfig.ErrorOutputPaths = []string{"stderr", kafkaURL.String()}

	if kafkaBrokers != nil || len(kafkaBrokers) > 0 {
		err := zap.RegisterSink("kafka", func(_ *url.URL) (zap.Sink, error) {
			config := sarama.NewConfig()
			config.Producer.Return.Successes = true

			producer, err := sarama.NewSyncProducer(kafkaBrokers, config)
			if err != nil {
				log.Fatal("failed to create kafka producer", err)
				return kafkaSink{}, err
			}

			return kafkaSink{
				producer: producer,
				topic:    "application.logs",
			}, nil
		})
		if err != nil {
			log.Fatal("failed to register kafka sink", err)
		}
	}

	zapLogger, err := zapConfig.Build(
		zap.AddCaller(),
		zap.AddStacktrace(zapcore.ErrorLevel),
	)
	if err != nil {
		log.Fatal("failed to build zap logger", err)
	}

	return zapLogger
}

func parseLevel(lvl string) (zapcore.Level, error) {
	switch strings.ToLower(lvl) {
	case "debug":
		return zap.DebugLevel, nil
	case "info":
		return zap.InfoLevel, nil
	case "warn":
		return zap.WarnLevel, nil
	case "error":
		return zap.ErrorLevel, nil
	}
	return zap.InfoLevel, fmt.Errorf("invalid log level <%v>", lvl)
}
```

We are covering the initial logger level, and registering kafka sink and default encoding congiruations. Just, keep in mind, there are some hard-coded parts like `application.logs` topic we will use it on kafka. You can define environment variables or applications configs for these kind of hard-coded values.

## Distributed Tracing with Tempo via OpenTelemetry

Tempo is open-source grafana's distributed tracing backend. It's quite simple and it can provide the tracing options like jaeger, zipkin, opentelemtry protocol etc. In this project, we will use the opentelemetry protocol to send the traces to tempo endpoint.

While defining the tracing we will use [otelfiber](https://github.com/gofiber/contrib/tree/main/otelfiber) library to integrate the opentelemetry with fiber framework. It propagates the tracing context over the fiber's UserContext. 

But at first we have to define our tracingprovider and exporter.

```go
exp, err := factory.NewOTLPExporter(ctx, cm.GetOTLPConfig().OTLPEndpoint)
if err != nil {
  log.Printf("failed to create OTLP exporter: %v", err)
  return
}

tp := factory.NewTraceProvider(exp, cm.GetAppConfig().Name)
defer func() { _ = tp.Shutdown(ctx) }()

otel.SetTracerProvider(tp)
otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(propagation.TraceContext{}, propagation.Baggage{}))
```

Above codeblock defines the opentelemetry tracerprovider and exporter.

Then, we can define the fiber middleware for tracing as follows;

```go
app.Use(
  otelfiber.Middleware(
  otelfiber.WithTracerProvider(
    a.TracerProvider,
  ),
  otelfiber.WithNext(func(c *fiber.Ctx) bool {
    return c.Path() == "/metrics"
  }),
))
```

As you see, we can skip the tracing for the `/metrics` endpoint. Because, it is not a business logic in our application.

## Setting up Simple Go Microservice with Fiber

We set a router handler with a simple http request to SWAPI[](https://swapi.dev/). We will use this handler to demonstrate the observability features.

```go
func (s *dataService) GetData(
	ctx context.Context,
	id string,
) (map[string]any, error) {
	span := trace.SpanFromContext(ctx)

	span.AddEvent("Starting fake long running task")

	sleepTime := 70
	sleepTimeDuration := time.Duration(sleepTime * int(time.Millisecond))
	time.Sleep(sleepTimeDuration)

	span.AddEvent("Done first fake long running task")

	span.AddEvent("Starting request to Star Wars API")

	s.logger.
		With(zap.String("traceID", span.SpanContext().TraceID().String())).
		Info("Getting data", zap.String("id", id))
	res, err := s.client.StarWars().GetPerson(ctx, id)
	if err != nil {
		span.RecordError(err)
		return nil, err
	}

	span.AddEvent("Done request to Star Wars API")

	return res, nil
}
```

In this code block, we defined the span events and added http request context to resty client to trace external requests as well.
With this approach, it is able to trace databases or any external services requests from our microservice.

Also, during logging we can add trace id via getting it from span context;
```go
span.SpanContext().TraceID().String()
```

It will give ability while examining the logs also we can query the traces with the same trace id.

## Exploring via Grafana Dashboard

Firstly, you can see the metrics on both Grafana and Prometheus metrics endpoint;

![explore-prometheus](/images/go-observability/prometheus-metrics.png)

Exploring the logs on Grafana via Loki datasource;

![explore-loki](/images/go-observability/loki-logs.png)

Exploring the traces on Grafana via Tempo datasource;

![explore-tempo](/images/go-observability/tempo-tracing-1.png)

Also you can see the much more detail on tempo dashboard;

![explore-tempo-detail](/images/go-observability/tempo-tracing-2.png)


## Source Code and How to run the Application

You can find the source code in [here](https://github.com/ilkerkorkut/go-examples/tree/master/microservice/observability).

To run the application dependencies like prometheus, loki, grafana, tempo and kafka, you can use the docker-compose file inside the container folder. But I have already covered the command in Makefile, so you can call the following command only;

Running depended services:

```shell
make local-up
```
1- Set the `/etc/hosts` file for the `kafka1`, `kafka2` and `kafka3` hostnames as `127.0.0.1` local ip.

2- `.vscode` folder contains the initial setup to debug golang application on your local machine. Just be sure that you have installed the `delve` debugger.

2.1- Other option; you can build the binary and run directly.(You have to set `.env` environment variables as well.)

```shell
make build
```