---
layout: post
title: "Distributed Tracing Across Services"
date: 2024-08-04
categories: observability
---

We had six services talking over HTTP and Kafka, and during incidents the hardest question was never "what broke?" — it was "what touched this request?" Logs existed per service. Metrics existed per host. Nothing tied them together. An incident that should have taken five minutes to resolve would burn an hour of manual log correlation.

This post covers how we wired up distributed tracing: correlation IDs first, then OpenTelemetry, then structured logging that actually made the whole thing queryable.

## The problem

![image from undraw](/assets/images/alert.jpg)

A user places an order. The request hits an API gateway, fans out to an inventory service, a payment service, a notification service, and eventually writes to a ledger via Kafka. Each service logs independently. When checkout latency spikes to 12 seconds, you're searching five different log streams by timestamp, hoping the clocks are synced and the log formats are consistent enough to piece together what happened.

We tried this for three months. The worst incident took 45 minutes to resolve — not because the fix was hard, but because finding the slow service required manually jumping between Kibana indices and guessing at timing overlaps.

## Correlation IDs: the quick win

Before going full OpenTelemetry, we started with the simplest possible thing: a UUID generated at the edge, propagated everywhere.

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

**Propagation rules we settled on:**
- HTTP calls: pass as `X-Correlation-ID` header (added to our shared RestTemplate config)
- Kafka messages: set in record headers, not the payload (avoids schema changes)
- Async workers: extract from message metadata before processing, put in MDC

This alone cut our mean-time-to-identify from ~30 minutes to ~8 minutes. One search by correlation ID across all indices would surface every log line for a transaction. The limitation: no timing information, no parent-child relationships, no visualization.

## OpenTelemetry: when correlation IDs aren't enough

We evaluated three options: Zipkin (lighter, less active development), Jaeger (mature, but self-hosted complexity), and OpenTelemetry with a managed backend. We went with OTel exporting to Grafana Tempo — mainly because we already had Grafana for metrics and didn't want another UI.

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

**Context propagation** is handled by W3C Trace Context headers (`traceparent`, `tracestate`). The OTel Java agent auto-instruments most HTTP clients and Kafka producers/consumers, so we only wrote manual spans for business-critical paths where we wanted custom attributes.

The trade-off we hit: the OTel Java agent adds ~50ms to startup and a small per-request overhead (~2ms in our measurements). For our latency budget this was fine. We considered the manual SDK-only approach (no agent) but decided the auto-instrumentation coverage was worth the overhead.

**Where we drew the line:** instrument at service boundaries only — incoming request, outgoing HTTP call, message publish, message consume. We explicitly decided not to instrument internal method calls. The signal-to-noise ratio drops fast when you trace everything.

## Structured logging

We had structured logging before tracing, but it wasn't connected. The missing piece was injecting trace context into every log line automatically.

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

We kept both the correlation ID (business identifier — the order ID) and the trace ID (infrastructure identifier). This lets you query from either direction: "show me everything for order 18473" or "show me everything in this trace." Different people ask different questions — support engineers think in orders, on-call engineers think in traces.

**Implementation (Logback with MDC):**

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

## What actually changed

The first incident after full rollout: alert fires for checkout latency at 2 AM. The alert itself now includes a trace ID (we configured AlertManager to attach it from the exemplar). Open Tempo, paste the trace ID, see the full waterfall. A downstream ledger service is taking 8 seconds on a database query — connection pool exhausted because a batch job was running during peak hours. Four minutes from alert to root cause.

Before tracing, that same incident pattern took 30-45 minutes. The fix was always simple once you found the slow service. The cost was in the finding.

The less obvious win: support tickets. "What happened to order #18473?" used to mean 20 minutes of log archaeology. Now it's one search. We built a small internal tool that takes an order ID, finds the correlation ID, pulls the trace, and renders the timeline. Support engineers use it directly without paging on-call.

**What I'd skip if doing it again:** we spent two weeks building custom dashboards for trace metrics (p99 per service, error rates by span). Grafana Tempo's built-in service graph and RED metrics dashboard gave us 90% of that for free. Should have started there.

**What I wouldn't skip:** keeping the correlation ID separate from the trace ID. Some teams just use the trace ID for everything. But traces are infrastructure-scoped — they break across async boundaries, batch jobs, retry queues. The business correlation ID survives all of that because it's just a string you propagate manually. When the Kafka consumer picks up a failed message three hours later, the trace is new but the correlation ID is the same.
