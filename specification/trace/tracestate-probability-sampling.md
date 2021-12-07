# TraceState: Probability Sampling

<!-- toc -->

- [Approach used in this document](#approach-used-in-this-document)
  * [Objective](#objective)
  * [Definitions](#definitions)
    + [Sampling](#sampling)
    + [Adjusted count](#adjusted-count)
    + [Sampler](#sampler)
    + [Parent-based sampler](#parent-based-sampler)
    + [Probability sampler](#probability-sampler)
    + [Consistent probability sampler](#consistent-probability-sampler)
    + [Trace completeness](#trace-completeness)
    + [Non-probability sampler](#non-probability-sampler)
    + [Always-on consistent probability sampler](#always-on-consistent-probability-sampler)
    + [Always-off sampler](#always-off-sampler)
- [Consistent Probability sampling](#consistent-probability-sampling)
  * [Conformance](#conformance)
  * [Completeness guarantee](#completeness-guarantee)
  * [Context invariants](#context-invariants)
    + [Sampled flag](#sampled-flag)
      - [Requirement: Inconsistent p-values are unset](#requirement-inconsistent-p-values-are-unset)
    + [P-value](#p-value)
      - [Requirement: Out-of-range p-values are unset](#requirement-out-of-range-p-values-are-unset)
    + [R-value](#r-value)
      - [Requirement: Out-of-range r-values unset both p and r](#requirement-out-of-range-r-values-unset-both-p-and-r)
      - [Requirement: R-value is generated with the correct probabilities](#requirement-r-value-is-generated-with-the-correct-probabilities)
  * [Samplers](#samplers)
    + [ParentConsistentProbabilityBased sampler](#parentconsistentprobabilitybased-sampler)
      - [Requirement: ParentConsistentProbabilityBased API](#requirement-parentconsistentprobabilitybased-api)
      - [Requirement: ParentConsistentProbabilityBased does not modify valid tracestate](#requirement-parentconsistentprobabilitybased-does-not-modify-valid-tracestate)
      - [Requirement: ParentConsistentProbabilityBased calls the configured root sampler for root spans](#requirement-parentconsistentprobabilitybased-calls-the-configured-root-sampler-for-root-spans)
      - [Requirement: ParentConsistentProbabilityBased respects the sampled flag for non-root spans](#requirement-parentconsistentprobabilitybased-respects-the-sampled-flag-for-non-root-spans)
    + [ConsistentProbabilityBased sampler](#consistentprobabilitybased-sampler)
      - [Requirement: TraceIdRatioBased API compatibility](#requirement-traceidratiobased-api-compatibility)
      - [Requirement: ConsistentProbabilityBased sampler sets r for root span](#requirement-consistentprobabilitybased-sampler-sets-r-for-root-span)
      - [Requirement: ConsistentProbabilityBased sampler unsets p when not sampling](#requirement-consistentprobabilitybased-sampler-unsets-p-when-not-sampling)
      - [Requirement: ConsistentProbabilityBased sampler sets p when sampling](#requirement-consistentprobabilitybased-sampler-sets-p-when-sampling)
      - [Requirement: ConsistentProbabilityBased sampler records unbiased adjusted counts](#requirement-consistentprobabilitybased-sampler-records-unbiased-adjusted-counts)
      - [Requirement: ConsistentProbabilityBased sampler sets r for non-root span](#requirement-consistentprobabilitybased-sampler-sets-r-for-non-root-span)
      - [Requirement: ConsistentProbabilityBased sampler acts like AlwaysOff sampler for probabilities less than 2**-62](#requirement-consistentprobabilitybased-sampler-acts-like-alwaysoff-sampler-for-probabilities-less-than-2-62)
  * [Composition rules](#composition-rules)
    + [List of requirements](#list-of-requirements)
      - [Requirement: Combining multiple sampling decisions using logical `or`](#requirement-combining-multiple-sampling-decisions-using-logical-or)
      - [Requirement: Combine multiple consistent probability samplers using the minimum p-value](#requirement-combine-multiple-consistent-probability-samplers-using-the-minimum-p-value)
      - [Requirement: Unset p when multiple consistent probability samplers decide not to sample](#requirement-unset-p-when-multiple-consistent-probability-samplers-decide-not-to-sample)
      - [Requirement: Use probability sampler p-value when its decision to sample is combined with non-probability samplers](#requirement-use-probability-sampler-p-value-when-its-decision-to-sample-is-combined-with-non-probability-samplers)
      - [Requirement: Use p-value 63 when a probability sampler decision not to sample is combined with a non-probability sampler decision to sample](#requirement-use-p-value-63-when-a-probability-sampler-decision-not-to-sample-is-combined-with-a-non-probability-sampler-decision-to-sample)
  * [Producer and consumer recommendations](#producer-and-consumer-recommendations)
    + [Trace producer: completeness](#trace-producer-completeness)
      - [Recommenendation: use non-descending power-of-two probabilities](#recommenendation-use-non-descending-power-of-two-probabilities)
    + [Trace producer: correctness](#trace-producer-correctness)
      - [Recommenendation: sampler delegation](#recommenendation-sampler-delegation)
    + [Trace producer: interoperability with `ParentBased` sampler](#trace-producer-interoperability-with-parentbased-sampler)
    + [Trace producer: interoperability with `TraceIDRatioBased` sampler](#trace-producer-interoperability-with-traceidratiobased-sampler)
    + [Trace consumer](#trace-consumer)
      - [Recommendation: Recognize inconsistent r-values](#recommendation-recognize-inconsistent-r-values)
  * [Appendix: Statistical test requirements](#appendix-statistical-test-requirements)
    + [Test procedure: non-powers of two](#test-procedure-non-powers-of-two)
      - [Requirement: Pass 15 non-power-of-two statistical tests](#requirement-pass-15-non-power-of-two-statistical-tests)
    + [Test procedure: exact powers of two](#test-procedure-exact-powers-of-two)
      - [Requirement: Pass 5 power-of-two statistical tests](#requirement-pass-5-power-of-two-statistical-tests)
    + [Test implementation](#test-implementation)

<!-- tocstop -->

**Status**: [Experimental](../document-status.md)

Probability sampling allows OpenTelemetry tracing users to lower their
collection costs with the use of randomized sampling techniques.
OpenTelemetry specifies how to convey and record the results of
probability sampling using the W3C `tracestate` in a way that allows
Span-to-Metrics pipelines to be built that accurately count sampled
spans.

## Approach used in this document

### Objective

This document specifies two `tracestate` fields, known as "r-value"
and "p-value" meant to support interoperable Sampler implementations.
Rules are given for creating, validating, interpreting, and mutating
these fields in an OpenTelemetry [context](../context/context.md).

Two Samplers are specified for optional inclusion in OpenTelemetry
tracing SDKs, named `ParentConsistentProbabilitityBased` and
`ConsistentProbabilityBased`.  These are meant as optional
replacements for the built-in `ParentBased` and `TraceIdRatioBased`
Samplers.

### Definitions

#### Sampling

Sampling is a family of techniques for collecting and analyzing only a
fraction of a complete data set.  Individual items that are "sampled"
are taken to represent one or more spans when collected and counted.
The representivity of each span is used in a Span-to-Metrics pipeline
to accurately count spans.

Sampling terminology uses "population" to refer to the complete set of
data being sampled from.  In OpenTelemetry tracing, "population"
refers to all spans.

In probability sampling, the representivity of individual sample items
is generally known, whereas OpenTelemetry also recognizes
"non-probability" sampling approaches, in which representivity is not
explicitly quantified.

#### Adjusted count

Adjusted count is a measure of representivity, the number of spans in
the population that are represented by the individually sampled span.
Span-to-metrics pipelines can be built by adding the adjusted count of
each sample span to a counter of matching spans.

For probability sampling, adjusted count is defined as the reciprocal
(i.e., mathematical inverse) of sampling probability.

For non-probability sampling, adjusted count is unknown.

Zero adjusted count is defined in a way that supports composition of
probability and non-probability sampling.  Zero is assigned as the
adjusted count when a probability sampler does not select a span.

#### Sampler

A Sampler provides configurable logic, used by the SDK, for selecting
which Spans are "recorded" and/or "sampled" in a tracing client
library.  To "record" a span means to build a representation of it in
the client's memory, which makes it eligible for being exported.  To
"sample" a span implies setting the W3C `sampled` flag, recording the
span, and exporting the span when it is finished.

OpenTelemetry supports spans that are "recorded" and not "sampled"
for in-process observability of live spans (e.g., z-pages).

The Sampler interface and the built-in Samplers defined by
OpenTelemetry decide immediately whether to sample a span, and the
child context immediately propagates the decision.

#### Parent-based sampler

A Sampler that makes its decision to sample based on the W3C `sampled`
flag from the context is said to use parent-based sampling.

#### Probability sampler

A probability Sampler is a Sampler that knows immediately, for each
of its decisions, the probability that the span had of being selected.

Sampling probability is defined as a number less than or equal to 1
and greater than 0 (i.e., `0 < probability <= 1`).  The case of 0
probability is treated as a special, non-probabilistic case.

#### Consistent probability sampler

A consistent probability sampler is a Sampler that supports
independent sampling decisions at each span in a trace while
maintaining that traces will be complete with a certain minimum
probability across the trace.

Consistent probability sampling requires that for any span in a given
trace, if a Sampler with lesser sampling probability selects the span
for sampling, then the span would also be selected by a Sampler
configured with greater sampling probability.

#### Trace completeness

A trace is said to be complete when all of the spans belonging to the
trace are collected.  When at least one span is collected but not all
spans are collected, the trace is considered incomplete.

Trace incompleteness may happen on purpose (e.g., through sampling
configuration), or by accident (e.g., through collection errors).  The
OpenTelemetry trace data model supports a _one-way_ test for
incompleteness: for any non-root span, the trace is definitely
incomplete if the span's parent span was not collected.

Incomplete traces that result from sampling configuration (i.e., on
purpose) are known as partial traces.  An important subset of the
partial traces are those which are also complete subtraces.  A
complete subtrace is defined at span S when every descendent span is
collected.

Since the test for an incompleteness is one-way, it is important to
know which sampling configurations may lead to incomplete traces.
Sampling configurations that lead naturally to complete traces and
complete subtraces are [discussed below](#trace-producer-completeness).

#### Non-probability sampler

A non-probability sampler is a Sampler that makes its decisions not
based on chance, but instead uses arbitrary logic and internal state.
The adjusted count of spans sampled by a non-probability sampler is
unknown.

#### Always-on consistent probability sampler

An always-on sampler is another name for a consistent probability
sampler with probability equal to one.

#### Always-off sampler

An always-off Sampler has the effect of disabling a span completely,
effectively excluding it from the population.  This is defined as a
non-probability sampler, not a zero-percent probability sampler,
because the spans are effectively unrepresented.

## Consistent Probability sampling

The consistent sampling scheme adopted by OpenTelemetry propagates two
values via the context, termed "p-value" and "r-value".

Both fields are propagated via the OpenTelemetry `tracestate` under
the `ot` vendor tag using the rules for [tracestate
handling](tracestate-handling.md).  Both fields are represented as
unsigned decimal integers requiring at most 6 bits of information.

This sampling scheme selects items from among a fixed set of 63
distinct probability values. The set of supported probabilities
includes the integer powers of two between 1 and 2**-62.

R-value encodes which among the 63 possibilities will consistently
decide to sample for a given trace.  Specifically, r-value specifies
the smallest probability that will decide to sample a given trace in
terms of the corresponding p-value.  For example, a trace with r-value
0 will sample spans configured for 100% sampling, while r-value 1 will
sample spans configured for 50% or 100% sampling, and so on through
r-value 62 which will sample every span.

P-value encodes the adjusted count for child contexts (i.e., consumers
of `tracestate`) and consumers of sampled spans to record for use in
Span-to-metrics pipelines.  A special p-value of 63 is defined to mean
zero adjusted count, which helps define composition rules for
non-probability samplers.

An invariant will be stated that connects the `sampled` trace flag
found in `traceparent` context to the r-value and p-value found in
`tracestate` context.

### Conformance

Consumers of OpenTelemetry `tracestate` data are expected to validate
the probability sampling fields before interpreting the data.  This
applies to the two samplers specified here as well as consumers of
span data, who are expected to validate `tracestate` before
interpreting span adjusted counts.

Producers of OpenTelemetry `tracestate` containing p-value and r-value
fields are required to meet the behavioral requirements stated for the
`ConsistentProbabilityBased` sampler and to ensure statistically valid
outcomes.  A test suite is included in this specification so that
users and consumers of OpenTelemetry `tracestate` can be assured of
accuracy in Span-to-metrics pipelines.

### Completeness guarantee

This specification defines consistent sampling for power-of-two
sampling probabilities.  When a sampler is configured with a
non-power-of-two sampling probability, the sampler will
probabilistically choose between the nearest powers of two.

When a single consistent probability sampler is used at the root of a
trace and all other spans use a parent-based sampler, the resulting
traces are always complete (ignoring collection errors).  This
property holds even for non-power-of-two sampling probabilities.

When multiple consistent probability samplers are used in the same
trace, in general, trace completeness is ensured at the smallest power
of two greater than or equal to the minimum sampling probability
across the trace.

### Context invariants

The W3C `traceparent` (version 0) contains three fields of
information: the TraceId, the SpanId, and the trace flags.  The
`sampled` trace flag has been defined by W3C to signal an intent to
sample the context.

The [Sampler API](sdk.md#sampler) is responsible for setting the
`sampled` flag and the `tracestate`.

P-value and r-value are set in the OpenTelemetry `tracestate`, under
the vendor tag `ot`, using the identifiers `p` and `r`.  P-value is an
unsigned integer valid in the inclusive range `[0, 63]` (i.e., there
are 64 valid values).  R-value is an unsigned integer valid in the
inclusive range `[0, 62]` (i.e., there are 63 valid values).  P-value
and r-value are independent settings, each can be meaningfully set
without the other present.

#### Sampled flag

Probability sampling uses additional information to enable consistent
decision making and to record the adjusted count of sampled spans.
When both values are defined and in the specified range, the invariant
between r-value and p-value and the `sampled` trace flag states that
`((p <= r) == sampled) OR (sampled AND (p == 63)) == TRUE`.

The invariant between `sampled`, `p`, and `r` only applies when both
`p` and `r` are present.  When the invariant is violated, the
`sampled` flag takes precedence and `p` is unset from `tracestate` in
order to signal unknown adjusted count.

##### Requirement: Inconsistent p-values are unset

Samplers SHOULD unset `p` when the invariant between the `sampled`,
`p`, and `r` values is violated before using the `tracestate` to make
a sampling decision or interpret adjusted count.

#### P-value

Zero adjusted count is represented by the special p-value 63,
otherwise the p-value is set to the negative base-2 logarithm of
sampling probability:

| p-value | Parent Probability | Adjusted count |
| -----   | -----------        | --             |
| 0       | 1                  | 1              |
| 1       | 1/2                | 2              |
| 2       | 1/4                | 4              |
| ...     | ...                | ...            |
| N       | 2**-N              | 2**N           |
| ...     | ...                | ...            |
| 61      | 2**-61             | 2**61          |
| 62      | 2**-62             | 2**62          |
| 63      | 0                  | 0              |

##### Requirement: Out-of-range p-values are unset

Consumers SHOULD unset `p` from the `tracestate` if the unsigned
decimal value is greater than 63 before using the `tracestate` to make
a sampling decision or interpret adjusted count.

#### R-value

R-value is set in the `tracestate` by the Sampler at the root of the
trace, in order to support consistent probability sampling.  When the
value is omitted or not present, child spans in the trace are not able
to participate in consistent probability sampling.

R-value determines which sampling probabilities and will not sample
for spans of a given trace, as follows:

| r-value          | Probability of r-value   | Implied sampling probabilities |
| ---------------- | ------------------------ | ----------------------         |
| 0                | 1/2                      | 1                              |
| 1                | 1/4                      | 1/2 and above                  |
| 2                | 1/8                      | 1/4 and above                  |
| 3                | 1/16                     | 1/8 and above                  |
| ...              | ...                      | ...                            |
| 0 <= r <= 61     | 2**-(r+1)                | 2**-r and above                |
| ...              | ...                      | ...                            |
| 59               | 2**-60                   | 2**-59 and above               |
| 60               | 2**-61                   | 2**-60 and above               |
| 61               | 2**-62                   | 2**-61 and above               |
| 62               | 2**-62                   | 2**-62 and above               |

These probabilities are specified to ensure that conforming Sampler
implementations record spans with correct adjusted counts.  The
recommended method of generating r-values is to count the number of
leading 0s in a string of 62 random bits, however it is not required
to use this approach.

##### Requirement: Out-of-range r-values unset both p and r

Samplers SHOULD unset both `r` and `p` from the `tracestate` if the
unsigned decimal value of `r` is greater than 62 before using the
`tracestate` to make a sampling decision.

##### Requirement: R-value is generated with the correct probabilities

Samplers MUST generate r-values using a randomized scheme that
produces each value with the specified probability.

#### Examples

Consider a trace context with the following headers:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: ot=r:3;p:2
```

The `traceparent` contents in this example example are repeated from
the [W3C specification](https://www.w3.org/TR/trace-context/#examples-of-http-traceparent-headers))
and have the following base64-encoded field values:

```
base16(version) = 00
base16(trace-id) = 4bf92f3577b34da6a3ce929d0e0e4736
base16(parent-id) = 00f067aa0ba902b7
base16(trace-flags) = 01  // (i.e., sampled)
```

The `tracestate` header contains OpenTelemetry string `r:3;p:2`,
containing decimal-encoded p-value and r-value:

```
base10(r) = 3
base10(p) = 2
```

Here, r-value 3 indicates that a consistent probability sampler
configured with probability 12.5% (i.e., 1-in-2**3) or greater will
sample the trace.  The p-value 2 indicates that the parent that set
the `sampled` flag was configured to sample at 25% (i.e., 1-in-2**2).
This trace context is consistent because `p <= r` is true and the
`sampled` flag is set.

Now, an example with an unsampled context where only the r-value is
set.

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-00
tracestate: ot=r:3
```

This supports consistent probability sampling in child contexts by
virtue of having an r-value, although p-value is not set, consistent
with an unsampled context.

### Samplers

#### ParentConsistentProbabilityBased sampler

The `ParentConsistentProbabilityBased` sampler is meant as an optional
replacement for the [`ParentBased` Sampler](sdk.md#parentbased). It is
required to first validate the `tracestate` and then respect the
`sampled` flag in the W3C traceparent.

##### Requirement: ParentConsistentProbabilityBased API

The `ParentConsistentProbabilityBased` Sampler constructor SHOULD take
a single Sampler argument, which is the Sampler to use in case the
`ParentConsistentProbabilityBased` Sampler is called for a root span.

##### Requirement: ParentConsistentProbabilityBased does not modify valid tracestate

The `ParentConsistentProbabilityBased` Sampler MUST NOT modify a
valid `tracestate`.

##### Requirement: ParentConsistentProbabilityBased calls the configured root sampler for root spans

The `ParentConsistentProbabilityBased` Sampler MUST delegate to the
configured root Sampler when there is not a valid parent trace context.

##### Requirement: ParentConsistentProbabilityBased respects the sampled flag for non-root spans

The `ParentConsistentProbabilityBased` Sampler MUST decide to sample
the span according to the value of the `sampled` flag in the W3C
traceparent header.

#### ConsistentProbabilityBased sampler

The `ConsistentProbabilityBased` sampler is meant as an optional
replacement for the [`TraceIdRatioBased`
Sampler](sdk.md#traceidratiobased).  In the case where it is used as a
root sampler, the `ConsistentProbabilityBased` sampler is required to
produce a valid `tracestate`.  In the case where it is used in a
non-root context, it is required to validate the incoming `tracestate`
and to produce a valid `tracestate` for the outgoing context.

The `ConsistentProbabilityBased` sampler is required to support
probabilities that are not exact powers of two.  To do so,
implementations are required to select between the nearest powers of
two probabilistically.  For example, 5% sampling can be achieved by
selecting 1/16 sampling 60% of the time and 1/32 sampling 40% of the
time.

##### Requirement: TraceIdRatioBased API compatibility

The `ConsistentProbabilityBased` Sampler MUST have the same
constructor signature as the built-in `TraceIdRatioBased` sampler in
each OpenTelemetry SDK.

##### Requirement: ConsistentProbabilityBased sampler sets r for root span

The `ConsistentProbabilityBased` Sampler MUST set `r` when it makes a
root sampling decision.

##### Requirement: ConsistentProbabilityBased sampler unsets p when not sampling

The `ConsistentProbabilityBased` Sampler MUST unset `p` from the
`tracestate` when it decides not to sample.

##### Requirement: ConsistentProbabilityBased sampler sets p when sampling

The `ConsistentProbabilityBased` Sampler MUST set `p` when it decides
to sample according to its configured sampling probability.

##### Requirement: ConsistentProbabilityBased sampler records unbiased adjusted counts

The `ConsistentProbabilityBased` Sampler MUST set `p` so that the
adjusted count interpreted from the `tracestate` is an unbiased
estimate of the number of representative spans in the population.

A test specification for this requirement is given in the appendix.

##### Requirement: ConsistentProbabilityBased sampler sets r for non-root span

If `r` is not set on the input `tracecontext` and the Span is not a
root span, `ConsistentProbabilityBased` SHOULD set `r` as if it were a
root span and MAY warn the user that a potentially inconsistent trace
is being produced.

##### Requirement: ConsistentProbabilityBased sampler acts like AlwaysOff sampler for probabilities less than 2**-62

If the configured sampling probability is less than `2**-62`, the
Sampler SHOULD round down to zero probability and make the same
sampling decision as the builtin `AlwaysOff` sampler would.

### Composition rules

When more than one Sampler participates in the decision to sample a
context, their decisions can be combined using composition rules.  In
all cases, the combined decision to sample is the logical-OR of the
Samplers' decisions (i.e., sample if at least one of the composite
Samplers decides to sample).

To combine p-values from two consistent probability Sampler decisions,
the Sampler with the greater probability takes effect.  The output
p-value becomes the minimum of the two values for `p`.

To combine a consistent probability Sampler decision with a
non-probability Sampler decision, p-value 63 is used to signify zero
adjusted count.  If the probability Sampler decides to sample, its
p-value takes effect.  If the probability Sampler decides not to
sample when the non-probability sample does sample, p-value 63 takes
effect signifying zero adjusted count.

#### List of requirements

##### Requirement: Combining multiple sampling decisions using logical `or`

When multiple samplers are combined using composition, the sampling
decision MUST be to sample if at least one of the combined samplers
decides to sample.

##### Requirement: Combine multiple consistent probability samplers using the minimum p-value

When combining Sampler decisions for multiple consistent probability
Samplers and at least one decides to sample, the minimum of the "yes"
decision `p` values MUST be set in the `tracestate`.

##### Requirement: Unset p when multiple consistent probability samplers decide not to sample

When combining Sampler decisions for multiple consistent probability
Samplers and none decides to sample, p-value MUST be unset in the
`tracestate`.

##### Requirement: Use probability sampler p-value when its decision to sample is combined with non-probability samplers

When combining Sampler decisions for a consistent probability Sampler
and a non-probability Sampler, and the probability Sampler decides to
sample, its p-value MUST be set in the `tracestate` regardless of the
non-probability Sampler decision.

##### Requirement: Use p-value 63 when a probability sampler decision not to sample is combined with a non-probability sampler decision to sample

When combining Sampler decisions for a consistent probability Sampler
and a non-probability Sampler, and the probability Sampler decides not
to sample but the non-probability does sample, p-value 63 MUST be set
in the `tracestate`.

### Producer and consumer recommendations

#### Trace producer: completeness

As stated in the [completeness guarantee](#completeness-guarantee),
traces will be possibly incomplete when configuring multiple
consistent probability samplers in the same trace.  One way to avoid
producing incomplete traces is to use parent-based samplers except for
root spans.

There is a simple test for trace incompleteness, but it is a one-way
test and does not detect when child spans are uncollected.  One way to
avoid producing incomplete traces is to avoid configuring
non-power-of-two sampling probabilities for non-root spans, because
completeness is not guaranteed for non-power-of-two sampling
probabilities.

##### Recommenendation: use non-descending power-of-two probabilities

Complete subtraces will be produced when the sequence of sampling
probabilities from the root of a trace to its leaves consists of
non-descending powers of two.  To ensure complete sub-traces are
produced, child samplers SHOULD be configured with a power-of-two
probability greater than or equal to the parent span's sampling
probability.

#### Trace producer: correctness

The use of tracestate to convey adjusted count information rests upon
trust between participants in a trace.  Users are advised not to use a
Span-to-metrics pipeline when the parent sampling decision's
corresponding adjusted count is untrustworthy.

The `ConsistentProbabilityBased` and
`ParentConsistentProbabilityBased` samplers can be used as delegates
of another sampler, for conditioning the choice of sampler on span and
other fixed attributes.  However, for adjusted counts to be
trustworthy, the choice of non-root sampler cannot be conditioned on
the parent's sampled trace flag or the OpenTelemetry tracestate
r-value and p-value, as these decisions would lead to incorrect
adjusted counts.

For example, the built-in [`ParentBased` sampler](sdk.md#parentbased)
supports configuring the delegated sampler based on whether the parent
context is remote or non-remote, sampled or unsampled.  If a
`ParentBased` sampler delegates to a `ConsistentProbabilityBased`
sampler only for unsampled contexts, the resulting Span-to-metrics
pipeline will (probably) overcount spans.

##### Recommenendation: sampler delegation

For non-root spans, composite samplers SHOULD NOT condition the choice
of delegating sampler based on the parent's sampled flag or
OpenTelemetry tracestate.

#### Trace producer: interoperability with `ParentBased` sampler

The OpenTelemetry built-in `ParentBased` sampler is interoperable with
the `ConsistentProbabilityBased` sampler, provided that the
delegated-to sampler does not change the decision that determined its
selection.  For example, it is safe to configure an alternate
`ParentBased` sampler delegate for unsampled spans, provided the
decision does not change to sampled.

Because the `ParentBased` sampler honors the sampled trace flag, and
OpenTelemetry SDKs include the tracestate in the `Span` data, which
means a system can be upgraded to probability sampling by just
replacing `TraceIDRatioBased` samplers with conforming
`ConsistentProbabilityBased` samplers everywhere in the trace.

#### Trace producer: interoperability with `TraceIDRatioBased` sampler

The [`TraceIDRatioBased` specification](sdk.md#traceidratiobased)
includes a RECOMMENDATION against being used for non-root spans
because it does not specify how to make the sampler decision
consistent across the trace.  A `TraceIDRatioBased` sampler at the
root span is interoperable with a `ConsistentParentProbabilityBased`
sampler in terms of completeness, although the resulting spans will
have unknown adjusted count.

When a `TraceIDRatioBased` sampler is configured for a non-root span,
several cases arise where an incorrect OpenTelemetry tracestate can be
generated.  Consider for example a trace with three spans where the
root (R) has a `ConsistentProbabilityBased` sampler, the root's child
(P) has a `TraceIDRatioBased` sampler, and the grand-child (C) has a
`ParentBased` sampler.  Because the `TraceIDRatioBased` sampler change
the intermediate sampled flag without updating the OpenTelemetry
tracestate, we have the following cases:

1. If `TraceIDRatioBased` does not change P's decision, the trace is
   complete and all spans' adjusted counts are correct.
2. If `TraceIDRatioBased` changes P's decision from no to yes, the
   consumer will observe a (definitely) incomplete trace containing P
   and C.  Both spans will have invalid OpenTelemetry tracestate,
   leading to unknown adjusted count in this case.
3. If `TraceIDRatioBased` changes the sampling decision from yes to
   no, the consumer will observe singleton trace with correct adjusted
   count. The consumer cannot determine that R has two unsampled
   descendents.

As these cases demonstrate, users can expect incompleteness and
unknown adjusted count when using `TraceIDRatioBased` samplers for
non-root spans, but this goes against the originally specified
warning.

#### Trace consumer

Trace consumers are expected to apply the simple one-way test for
incompleteness.  When non-root spans are configured with independent
sampling probabilities, traces may be complete in a way that cannot be
detected.  Because of the one-way test, consumers wanting to ensure
complete traces are expected to know the minimum sampling probability
across the system.

Ignoring accidental data loss, a trace will be complete if all its
spans are sampled with consistent probability samplers and the trace's
r-value is larger than the corresponding smallest power of two greater
than or equal to the minimum sampling probability across the trace.

Due to the `ConsistentProbabilityBased` Sampler requirement about
setting `r` when it is unset for a non-root span, trace consumers are
advised to check traces for r-value consistency.  When a single trace
contains more than a single distinct `r` value, it means the trace was
not correctly sampled at the root for probability sampling.  While the
adjusted count of each span is correct in this scenario, it may be
impossible to detect complete traces.

##### Recommendation: Recognize inconsistent r-values

When a single trace contains spans with `tracestate` values containing
more than one distinct value for `r`, the consumer SHOULD recognize
the trace as inconsistently sampled.

### Appendix: Statistical test requirements

This section specifies a test that can be implemented to ensure basic
conformance with the requirement that sampling decisions are unbiased.

The goal of this test specification is to be simple to implement and
not require advanced statistical skills or libraries to be successful.

This test is not meant to evaluate the performance of a random number
generator.  This test assumes the underlying RNG is of good quality
and checks that the sampler produces the expected proportionality with
a high degree of statistical confidence.

One of the challenges of this kind of test is that probabilistic tests
are expected to occasionally produce exceptional results.  To make
this a strict test for random behavior, we take the following approach:

- Generate a pre-determined list of 20 random seeds
- Use fixed values for significance level (5%) and trials (20)
- Use a population size of one million spans
- For each trial, simulate the population and compute ChiSquared
  test statistic
- Locate the first seed value in the ordered list such that the
  Chi-Squared significance test fails exactly once out of 20 trials

To create this test, perform the above sequence using the seed values
from the predetermined list, in order, until a seed value is found
with exactly one failure.  This is expected to happen fairly often and
is required to happen once among the 20 available seeds.  After
calculating the index of the first seed with exactly one ChiSquared
failure, record it in the test.  For continuous integration testing,
it is only necessary to re-run the test using the predetermined seed
index.

As specified, the Chi-Squared test has either one or two degrees of
freedom, depending on whether the sampling probability is an exact
power of two or not.

#### Test procedure: non-powers of two

In this case there are two degrees of freedom for the Chi-Squared test.
The following table summarizes the test parameters.

| Test case | Sampling probability | Lower, Upper p-value when sampled | Expect<sub>lower</sub> | Expect<sub>upper</sub> | Expect<sub>unsampled</sub> |
| ---       | ---                  | ---                               | ---                    | ---                    | ---                        |
| 1         | 0.900000             | 0, 1                              | 100000                 | 800000                 | 100000                     |
| 2         | 0.600000             | 0, 1                              | 400000                 | 200000                 | 400000                     |
| 3         | 0.330000             | 1, 2                              | 170000                 | 160000                 | 670000                     |
| 4         | 0.130000             | 2, 3                              | 120000                 | 10000                  | 870000                     |
| 5         | 0.100000             | 3, 4                              | 25000                  | 75000                  | 900000                     |
| 6         | 0.050000             | 4, 5                              | 12500                  | 37500                  | 950000                     |
| 7         | 0.017000             | 5, 6                              | 14250                  | 2750                   | 983000                     |
| 8         | 0.010000             | 6, 7                              | 5625                   | 4375                   | 990000                     |
| 9         | 0.005000             | 7, 8                              | 2812.5                 | 2187.5                 | 995000                     |
| 10        | 0.002900             | 8, 9                              | 1006.25                | 1893.75                | 997100                     |
| 11        | 0.001000             | 9, 10                             | 953.125                | 46.875                 | 999000                     |
| 12        | 0.000500             | 10, 11                            | 476.5625               | 23.4375                | 999500                     |
| 13        | 0.000260             | 11, 12                            | 228.28125              | 31.71875               | 999740                     |
| 14        | 0.000230             | 12, 13                            | 14.140625              | 215.859375             | 999770                     |
| 15        | 0.000100             | 13, 14                            | 22.0703125             | 77.9296875             | 999900                     |

The formula for computing Chi-Squared in this case is:

```
ChiSquared = math.Pow(sampled_lowerP - expect_lowerP, 2) / expect_lowerP +
             math.Pow(sampled_upperP - expect_upperP, 2) / expect_upperP +
             math.Pow(1000000 - sampled_lowerP - sampled_upperP - expect_unsampled, 2) / expect_unsampled
```

This should be compared with 0.102587, the value of the Chi-Squared
distribution for two degrees of freedom with significance level 5%.
For each probability in the table above, the test is required to
demonstrate a seed that produces exactly one ChiSquared value less
than 0.102587.

##### Requirement: Pass 15 non-power-of-two statistical tests

For the test with 20 trials and 1 million spans each, the test MUST
demonstrate a random number generator seed such that the ChiSquared
test statistic is below 0.102587 exactly 1 out of 20 times.

#### Test procedure: exact powers of two

In this case there is one degree of freedom for the Chi-Squared test.
The following table summarizes the test parameters.

| Test case | Sampling probability | P-value when sampled | Expect<sub>sampled</sub> | Expect<sub>unsampled</sub> |             |
| ---       | ---                  | ---                  | ---                      | ---                        |             |
| 16        | 0x1p-01 (0.500000)   | 1                    | 500000                   | n/a                        | 500000      |
| 17        | 0x1p-04 (0.062500)   | 4                    | 62500                    | n/a                        | 937500      |
| 18        | 0x1p-07 (0.007812)   | 7                    | 7812.5                   | n/a                        | 992187.5    |
| 19        | 0x1p-10 (0.000977)   | 10                   | 976.5625                 | n/a                        | 999023.4375 |
| 20        | 0x1p-13 (0.000122)   | 13                   | 122.0703125              | n/a                        | 999877.9297 |

The formula for computing Chi-Squared in this case is:

```
ChiSquared = math.Pow(sampled - expect_sampled, 2) / expect_sampled +
             math.Pow(1000000 - sampled - expect_unsampled, 2) / expect_unsampled
```

This should be compared with 0.003932, the value of the Chi-Squared
distribution for one degree of freedom with significance level 5%.
For each probability in the table above, the test is required to
demonstrate a seed that produces exactly one ChiSquared value less
than 0.003932.

##### Requirement: Pass 5 power-of-two statistical tests

For the teset with 20 trials and 1 million spans each, the test MUST
demonstrate a random number generator seed such that the ChiSquared
test statistic is below 0.003932 exactly 1 out of 20 times.

#### Test implementation

The recommended structure for this test uses a table listing the 20
probability values, the expected p-values, whether the ChiSquared
statistic has one or two degrees of freedom, and the index into the
predetermined list of seeds.

```
    for _, test := range []testCase{
        // Non-powers of two
        {0.90000, 1, twoDegrees, 5},
        {0.60000, 1, twoDegrees, 14},
        {0.33000, 2, twoDegrees, 3},
        {0.13000, 3, twoDegrees, 2},
        {0.10000, 4, twoDegrees, 0},
        {0.05000, 5, twoDegrees, 0},
        {0.01700, 6, twoDegrees, 2},
        {0.01000, 7, twoDegrees, 3},
        {0.00500, 8, twoDegrees, 1},
        {0.00290, 9, twoDegrees, 1},
        {0.00100, 10, twoDegrees, 5},
        {0.00050, 11, twoDegrees, 1},
        {0.00026, 12, twoDegrees, 3},
        {0.00023, 13, twoDegrees, 0},
        {0.00010, 14, twoDegrees, 2},

        // Powers of two
        {0x1p-1, 1, oneDegree, 0},
        {0x1p-4, 4, oneDegree, 2},
        {0x1p-7, 7, oneDegree, 3},
        {0x1p-10, 10, oneDegree, 0},
        {0x1p-13, 13, oneDegree, 1},
    } {
```

Note that seed indexes in the example above have what appears to be
the correct distribution.  The five 0s, four 1s, four 2s, four 3s, and
two 5s demonstrate that it is relatively easy to find examples where
there is exactly one failure.  Seed index 14, for probability 0.6 in
this case, is a reminder that outliers exist.  Further significance
testing of this distribution is not recommended.