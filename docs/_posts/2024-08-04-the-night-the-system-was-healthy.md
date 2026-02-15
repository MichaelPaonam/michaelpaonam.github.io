---
layout: post
---
# An Observability Story

![image from undraw](/assets/images/alert.jpg)

02:13 AM.

I received an alert.

> **Checkout latency > 8 seconds**

I opened Grafana. And I found that the CPUs were normal, Memory was stable and Database was calm. Nothing seemed to be broken. And yet, the customers had to wait 8 seconds to place an order

So, I went to the logs to investigate further.

* Service A: fine
* Service B: fine
* Service C: retrying
* Service D: timing out
* Service E: never received the request

The system wasn’t failing. It was disagreeing with itself.

---

## The First False Assumption

The first mistake we made was treating incidents like component failures. But distributed systems don’t fail like machines. They fail like conversations.

Some services heard the message.
Some heard it twice.
Some heard it late.
Some never heard it.

We were trying to debug machines. When we should be looking at the history.

*"Observability exists to reconstruct that history — correlating signals to understand why something failed."*

---

## Then came the support ticket

A support engineer asks:

> “Can you find what happened to order #18473?”

We couldn't track the order.

We could search for:

* host
* service
* time range

But not the *event itself*.

Because the system was tracking infrastructure — not intent.

---

## The Missing Thread

After hours of tears and sweat, we finally found our solution - "distributed tracing".
A distributed trace tracks execution paths across services. A correlation ID binds events into one logical transaction.

So, in the next planning meeting, we added a new rule:

> Every business action (transaction) must carry an identity (which became the correlation ID).

Not request identity.
Not span identity.

**Business identity.**

Order-ID propagated through HTTP, Kafka, retries, batch repair jobs.

---

## The Next Incident

03:02 AM.

PagerDuty again.

Latency spike.

Search logs:

```
order=18473
```

One query.
Seven services.
Three retries.
One slow downstream ledger.

Incident resolved in four minutes 😎.

---

## Lessons learned
Telemetry without causality is noise. Once causality exists, debugging becomes reading. Systems rarely hide information. They hide relationships.

And relationships are the only thing operators actually need.
