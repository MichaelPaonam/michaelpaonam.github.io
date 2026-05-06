---
layout: post
title: "Distributed Tracing Across Services"
date: 2024-08-04
categories: observability
---

When you have five or six services talking over HTTP and Kafka, the hardest question during an incident isn't "what broke?" — it's "what touched this request?" Logs exist per service. Metrics exist per host. But nothing ties them together unless you explicitly build that connective tissue.

This post covers how we set up distributed tracing: correlation IDs, OpenTelemetry instrumentation, and structured logging that actually lets you follow a transaction end-to-end.

## The problem

![image from undraw](/assets/images/alert.jpg)

A user places an order. That request hits an API gateway, fans out to an inventory service, a payment service, a notification service, and eventually writes to a ledger via Kafka. Each service logs independently. When something goes wrong — say a 12-second checkout — you're searching five different log streams by timestamp, hoping the clocks are synced and the log formats are consistent enough to correlate manually.

This doesn't scale. You need a shared identifier that travels with the request.

## Correlation IDs

The simplest starting point: generate a unique ID at the edge and propagate it through every service that handles the request.

**At the gateway (Spring Boot filter):**

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String correlationId = request.getHeader("X-Correlation-ID");
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }
        MDC.put("correlationId", correlationId);
        response.setHeader("X-Correlation-ID", correlationId);
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove("correlationId");
        }
    }
}
```

**Propagation rules:**
- HTTP calls: pass as `X-Correlation-ID` header
- Kafka messages: set in message headers (not the payload)
- Async workers: extract from the message metadata before processing

Every log line includes this ID. That alone gets you from "search by timestamp" to "search by transaction."

## OpenTelemetry

Correlation IDs solve log correlation, but they don't give you timing, causality, or a visual trace. OpenTelemetry gives you spans — units of work with start/end times, parent-child relationships, and attributes.

**Basic setup (Spring Boot with OTel SDK):**

```java
@Configuration
public class TracingConfig {

    @Bean
    public OpenTelemetry openTelemetry() {
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(
                OtlpGrpcSpanExporter.builder().build()
            ).build())
            .build();

        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .setPropagators(ContextPropagators.create(
                W3CTraceContextPropagator.getInstance()
            ))
            .build();
    }
}
```

**Instrumenting a service call:**

```java
@Service
public class PaymentService {

    private final Tracer tracer;

    public PaymentService(OpenTelemetry openTelemetry) {
        this.tracer = openTelemetry.getTracer("payment-service");
    }

    public PaymentResult processPayment(String orderId, String method, long amount) {
        Span span = tracer.spanBuilder("process_payment").startSpan();
        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("order.id", orderId);
            span.setAttribute("payment.method", method);
            PaymentResult result = paymentClient.charge(orderId, amount);
            span.setAttribute("payment.status", result.getStatus());
            return result;
        } finally {
            span.end();
        }
    }
}
```

**Context propagation** is handled by W3C Trace Context headers (`traceparent`, `tracestate`). Most HTTP client libraries have OTel instrumentation packages that inject these automatically. For Kafka, you propagate the trace context in message headers and extract it on the consumer side.

The key architectural decision: instrument at service boundaries (incoming request, outgoing call, message publish, message consume). You don't need to instrument every function — just the points where execution crosses a network boundary.

## Structured logging

Unstructured logs (`INFO: payment processed for order 18473`) are human-readable but machine-hostile. Structured logs let you filter and aggregate:

```json
{
  "timestamp": "2024-07-15T02:13:47Z",
  "level": "info",
  "service": "payment-service",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "correlation_id": "ord-18473-a8f2",
  "message": "payment processed",
  "order_id": "18473",
  "amount_cents": 4500,
  "duration_ms": 230
}
```

The trace ID and span ID come from the active OpenTelemetry context. The correlation ID is your business-level identifier. Having both means you can query from either direction: start from the trace (infrastructure view) or start from the order (business view).

**Implementation pattern (Logback with MDC):**

```xml
<!-- logback-spring.xml -->
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>correlationId</includeMdcKeyName>
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
    </encoder>
</appender>
```

```java
@Component
public class TraceContextLogger implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        Span span = Span.current();
        SpanContext ctx = span.getSpanContext();
        if (ctx.isValid()) {
            MDC.put("traceId", ctx.getTraceId());
            MDC.put("spanId", ctx.getSpanId());
        }
        return true;
    }
}
```

## What changes operationally

Before: an alert fires for high checkout latency. You open Grafana, confirm it's not CPU or memory, then start grepping logs across services by timestamp. You find the slow service after 20 minutes of manual correlation.

After: the alert includes a trace ID. You open Jaeger or Tempo, paste the trace ID, and see the full request waterfall — which services were called, in what order, and where the time was spent. A downstream ledger service took 8 seconds on a database query. Incident resolved in minutes, not hours.

The investment is real — instrumenting services, agreeing on propagation conventions, setting up a trace collector and backend. But the payoff compounds with every incident, every support ticket, and every "what happened to order X?" question that used to take an hour to answer.
