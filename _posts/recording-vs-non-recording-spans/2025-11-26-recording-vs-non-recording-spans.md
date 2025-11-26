---
title: The Mystery of the Missing Data
date: 2025-11-26
tags: ['python', 'observability', 'programming']
description: A Journey Through OpenTelemetry Performance
---


It was 2 AM on a Tuesday when I got the call. Our production microservices were experiencing mysterious performance degradation, and the observability team was baffled. Traces were showing up for some requests but not others, and when they did appear, they were incomplete. The worst part? The CPU usage was through the roof.

I had been working with OpenTelemetry for about six months at that point, integrating it across our Python microservices architecture. We run a platform processing millions of requests daily, so every millisecond matters. That night, I learned a lesson that fundamentally changed how I think about distributed tracing.

### The Setup: What We Were Doing

Our system had three main services:

1. **API Gateway** - Entry point for all requests
2. **Auth Service** - Validates user credentials
3. **Data Service** - Processes and stores data

We had configured OpenTelemetry with what I thought was a sensible setup:
- `ProbabilitySampler` set to 10% on the API Gateway (sample 1 in 10 requests)
- `AlwaysOnSampler` on the Auth and Data services (sample everything)

The reasoning seemed sound: we don't want to process every trace, so sample at the entry point and let downstream services decide independently. Simple, right? Wrong.

### The Problem Reveals Itself

I pulled up the Jaeger dashboard and noticed something odd. When the API Gateway sampled a request (the 10% that passed through), the downstream services were creating spans, but they were being created as **NonRecording spans**. These spans had zero effect—they didn't show up in our traces, they didn't export data, and they didn't contribute to the trace we wanted to analyze.

But here's the catch: these NonRecording spans were still doing *something*. Each one was still being created, still propagating context, still going through span processors. It felt like we were paying a cost for spans we weren't even using.

I started digging into the OpenTelemetry specification and the Python SDK source code. That's when I discovered the distinction that would change everything.

### Recording Spans vs. NonRecording Spans: The Epiphany

**Recording Spans** are the spans you actually want. They:
- Capture all your `SetAttribute()` calls, `AddEvent()` calls, and status information
- Get processed by span processors and eventually exported to Jaeger, Datadog, or wherever
- Have a meaningful overhead because they're storing real data
- Show up in your traces and dashboards

```python
# A Recording Span - everything gets captured
with tracer.start_as_current_span("database_query") as span:
    span.set_attribute("query_type", "SELECT")
    span.set_attribute("table", "users")
    span.add_event("query_started")
    # ... do work ...
    span.set_attribute("rows_returned", 42)
```

**NonRecording Spans**, on the other hand, are the spans that don't make the cut:
- The sampler decided they shouldn't be recorded
- All calls to `SetAttribute()`, `AddEvent()`, etc. are instant no-ops
- They're never exported anywhere
- They still propagate trace context to child services (crucial for distributed tracing)
- They should be virtually free from a performance perspective

But here's where I got it wrong: I was treating NonRecording spans like they cost nothing, when in reality, I was *creating a NonRecording span for every single request* even though the parent didn't get recorded.

### The Ah-Ha Moment

The issue was in how I'd configured the samplers. The API Gateway sampled at 10%, which meant 90% of requests got NonRecording spans. Those NonRecording spans then propagated the "don't record this" decision downstream through the trace context headers.

When the Auth Service received a request without the "record" flag, even though it was configured with `AlwaysOnSampler`, it respected the parent's decision (thanks to `ParentBasedSampler` being the default behavior). So it created a NonRecording span too.

The cascade continued to the Data Service. Same result.

So here's what was happening:
1. Request arrives at API Gateway
2. 90% of the time: NonRecording span created (no-op)
3. Request goes to Auth Service
4. Auth Service sees "don't record this trace" in the context
5. NonRecording span created (no-op)
6. Request goes to Data Service
7. Same story: NonRecording span created

Three NonRecording spans per request, 90% of the time. Millions of requests per day. That's a lot of no-ops.

But wait—NonRecording spans are supposed to be free. So why was CPU spiking?

### The Real Culprit: Premature Optimization Gone Wrong

Then I looked at my span processor code. That was the problem. I had written a custom span processor that was doing *everything* unconditionally:

```python
class MyCustomSpanProcessor(SpanProcessor):
    def on_start(self, span, parent_context=None):
        # I was doing expensive operations on EVERY span
        detailed_info = get_detailed_system_info()  # OUCH!
        user_context = lookup_user_from_database()   # OUCH!
        span.set_attribute("system_info", detailed_info)
        span.set_attribute("user", user_context)
    
    def on_end(self, span):
        # More expensive work
        pass
```

The problem: this processor was running on NonRecording spans too! Even though the span would never be exported, my code was computing expensive information for every single one.

It took me a few hours of profiling to realize that most of the CPU cost was coming from this processor, not from the span creation itself.

### The Fix: One Simple Check

The fix was almost embarrassingly simple. I just needed to check if the span was recording before doing expensive work:

```python
class MyCustomSpanProcessor(SpanProcessor):
    def on_start(self, span, parent_context=None):
        # Only do expensive operations if we're actually recording
        if span.is_recording():
            detailed_info = get_detailed_system_info()
            user_context = lookup_user_from_database()
            span.set_attribute("system_info", detailed_info)
            span.set_attribute("user", user_context)
    
    def on_end(self, span):
        if span.is_recording():
            # Only process spans that will actually be exported
            pass
```

But there's more. I also added a guard in my instrumented code:

```python
@app.route("/api/users/<user_id>")
def get_user(user_id):
    with tracer.start_as_current_span("fetch_user_details") as span:
        # Only compute expensive data if we're recording
        if span.is_recording():
            user_data = fetch_from_database(user_id)
            span.set_attribute("user_data", serialize(user_data))
        else:
            # Still fetch data for the response, just don't add to span
            user_data = fetch_from_database(user_id)
        
        return user_data
```

I deployed this fix on a Tuesday afternoon. By Wednesday morning, CPU usage had dropped by 35%. Let me repeat that: **35% CPU reduction** just by adding a single `if span.is_recording()` check.

### What Changed

Here's what I learned that night (well, morning... it was a long debugging session):

1. **NonRecording spans are truly free if you treat them correctly** - If you don't do anything in your code that requires actually recording data, a NonRecording span has virtually zero overhead. The span context propagation is handled at the framework level.

2. **The sampler makes the decision at span creation time** - You don't get to decide later whether a span is recording. The sampler decides when the span is created based on your sampling strategy and parent context.

3. **Sampling strategy matters downstream** - Using `ParentBasedSampler` (the default) means downstream services respect the parent's sampling decision. This ensures consistency but also means a single sampling decision at your entry point affects the entire trace tree.

4. **Check before expensive operations** - Always guard expensive operations with `span.is_recording()`. This is a pattern I now follow religiously.

5. **NonRecording spans still propagate context** - Even though they're not recorded, they still pass trace context to child services. This is essential for distributed tracing to work correctly across your system.

### The Architecture Lesson

Looking back, I realized my sampling strategy was actually the real issue. I was trying to do sampling at the entry point, but I hadn't considered the implications downstream.

Here's what I changed:

**Before (naive approach):**
- API Gateway: 10% sample rate
- Auth Service: Always sample (but respects parent)
- Data Service: Always sample (but respects parent)
- Result: 90% of traces were silently not recorded, leading to suspicious NonRecording spans everywhere

**After (thoughtful approach):**
- API Gateway: 100% sample rate (it's the entry point; we make the decision here)
- Auth Service: Respect parent (no sampler specified, uses default ParentBasedSampler)
- Data Service: Respect parent (same as above)
- Result: Clear recording decisions, no unnecessary NonRecording spans, cleaner mental model

OR, if we really wanted 10% sampling:

- API Gateway: 10% sample rate
- Auth Service: 10% sample rate (not respect parent, be explicit)
- Data Service: 10% sample rate (not respect parent, be explicit)
- Result: Consistent sampling across all services, no surprises from NonRecording spans

### The Lesson Sticks

That incident taught me that OpenTelemetry's design, while powerful, requires you to understand the underlying concepts deeply. NonRecording spans aren't a flaw—they're a feature. They allow efficient sampling by creating no-op spans that still propagate context. But you need to know how to work with them.

Now, whenever I instrument code, I follow these principles:

1. **Understand your sampler** - Know why spans are being sampled the way they are
2. **Use `is_recording()` as a guard** - Before expensive operations, check if the span is recording
3. **Configure samplers consciously** - Think about where you want sampling decisions made
4. **Profile before and after** - Don't assume anything; measure the impact

### Code Example: The Right Way

Here's the pattern I now use in all my instrumented code:

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def process_request(request):
    with tracer.start_as_current_span("process_request") as span:
        # Cheap operations - always do these
        span.set_attribute("request_id", request.id)
        span.set_attribute("method", request.method)
        
        # Expensive operations - guard with is_recording()
        if span.is_recording():
            span.set_attribute("request_headers", dict(request.headers))
            span.set_attribute("user_context", get_user_context(request))
        
        # Your business logic
        result = do_work(request)
        
        # Record result only if we're sampling
        if span.is_recording():
            span.set_attribute("result_status", result.status)
            span.set_attribute("processing_time_ms", result.time_ms)
        
        return result
```

### Conclusion

OpenTelemetry's distinction between recording and non-recording spans reflects a deep understanding of observability at scale. It's not enough to just add tracing to your code; you need to understand *how* tracing decisions cascade through your system.

That 2 AM incident, the mysterious CPU spike, and the hours spent debugging—they all led me to appreciate the elegance of this design. NonRecording spans let you sample efficiently without sacrificing distributed trace context propagation. But you have to use them wisely.

If you're working with OpenTelemetry in Python, I'd recommend taking the time to understand recording vs. non-recording spans deeply. Add some profiling to see where the overhead is coming from. And always, *always* check `span.is_recording()` before doing expensive work.

Your CPU (and your SRE team) will thank you.
