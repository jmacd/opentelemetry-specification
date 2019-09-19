# Metrics API

## Overview

The _Metrics API_ supports reporting diagnostic measurements using
three basic kinds of instrument, commonly known as Counters, Gauges,
and Measures.  Instruments are code-level objects, handled by
programmers, used to gain visibility into operational metrics about
the application, its components, and the system as a whole.

Monitoring and alerting are the common use-case for the data provided
through metric instruments, after various collection and aggregation
strategies are applied to the data.  We find there are many other uses
for the _metric events_ that stream into these instruments.  We
imagine metric data being aggregated and recorded as events in tracing
and logging systems too, and for this reason OpenTelemetry requires a
separation of the API from the SDK.

### Meter

The OpenTelemetry API that provides the metrics SDK is named `Meter`.
According to the specification, the `Meter` implementation ultimately
determines how metrics events are handled.  The specification's task
is to define the semantics of the event and describe standard
interpretation in high-level terms.  How the `Meter` accomplishes its
goals and the export capabilities it supports are not specified.

The standard interpretation for `Meter` implementations to follow is
specified so that users understand the intended use for each kind of
metric.  For example, a monotonic Counter instrument supports
`Add()` events, so the standard interpretation is to compute a sum;
the sum may be exported as an absolute value or as the change in
value, but either way the purpose of using a Counter with `Add()` is
to monitor a sum.

## Metric kinds and inputs

The API distinguishes metric instruments by semantic meaning, not by
the type of value produced in an exporter.  This is a departure from
convention, compared with a number of common metric libraries, and
stems from the separation of the API and the SDK.  The SDK ultimately
determines how to handle metric events and could potentially implement
non-standard behavior.

This explains why the metric API does not have metric instrument kinds
for exporting "Histogram" and "Summary" distribution explicitly, for
example.  These are both semantically `Measure` instruments and an SDK
can be configured to produce histograms or distribution summaries from
Measure events.  It is out of scope for the Metrics API to specify how
these alternatives are configured in a particular SDK.

We believe the three metric kinds Counter, Gauge, and Measure form a
sufficient basis for expression of a wide variety of metric data.
Programmers write and read these as `Add()`, `Set()`, and `Record()`
method calls, signifying the semantics and standard interpretation,
and we believe these three methods are all that are needed.

Nevertheless, it is common to apply restrictions on metric values, the
inputs to `Add()`, `Set()`, and `Record()`, in order to refine their
standard interpretation.  Generally, there is a question of whether
the instrument can be used to compute a rate, because that is usually
a desireable analysis.  Each metric instrument offers an optional
declaration, specifying restrictions on values input to the metric.
For example, Measures are declared as non-negative by default,
appropriate for reporting sizes and durations; a Measure option is
provided to record positive or negative values, but it does not change
the kind of instrument or the method name used, as the semantics are
unchanged.

### Metric selection

To guide the user in selecting the right kind of metric for an
application, we'll consider the following questions about the primary
intent of reporting given data. We use "of primary interest" here to
mean information that is almost certainly useful in understanding
system behavior. Consider these questions:

- Does the measurement represent a quantity of something? Is it also non-negative?
- Is the sum a matter of primary interest?
- Is the event count a matter of primary interest?
- Is the distribution (p50, p99, etc.) a matter of primary interest?

With answers to these questions, a user should be able to select the
kind of metric instrument based on its primary purpose.

### Counter

Counters support `Add(value)`.  Choose this kind of metric when the
value is a quantity, the sum is of primary interest, and the event
count and value distribution are not of primary interest.

Counters are defined as monotonic by default, meaning that positive
values are expected.  Monotonic counters are typically used because
they can automatically be interpreted as a rate.

As an option, counters can be declared as `NonMonotonic`, in which
case they support positive and negative increments.  Non-monotonic
counters are useful to report changes in an accounting scheme, such as
the number of bytes allocated and deallocated.

### Gauge

Gauges support `Set(value)`.  Gauge metrics express a pre-calculated
value that is either Set() by explicit instrumentation or observed
through a callback.  Generally, this kind of metric should be used
when the metric cannot be expressed as a sum or because the
measurement interval is arbitrary. Use this kind of metric when the
measurement is not a quantity, and the sum and event count are not of
interest.

Gauges are defined as non-monotonic by default, meaning that any value
(positive or negative) is allowed.

As an option, gauges can be declared as `Monotonic`, in which case
successive values are expected to rise monotonically.  Monotonic
gauges are useful in reporting computed cumulative sums, allowing an
application to compute a current value and report it, without
remembering the last-reported value in order to report an increment.

A special case of gauge is supported, called an `Observer` metric
instrument, which is semantically equivalent to a gauge but uses a
callback to report the current value.  Observer instruments are
defined by a callback, instead of supporting `Set()`, but the
semantics are the same.  The only difference between `Observer` and
ordinary gauges is that their events do not have an associated
OpenTelemetry context.  Observer instruments are non-monotonic by
default and monotonic as an option, like ordinary gauges.

### Measure

Measures support `Record(value)`, signifying that events report
individual measurements.  This kind of metric should be used when the
count or rate of events is meaningful and either:

- The sum is of interest in addition to the count (rate)
- Quantile information is of interest.

Measures are defined as `NonNegative` by default, meaning that
negative values are invalid.  Non-negative measures are typically used
to record absolute values such as durations and sizes.

As an option, measures can be declared as `Signed` to indicate support
for positive and negative values.

## Detailed Description

### Structure

Metric instruments are named.  Regardless of the instrument kind,
metric events include the instrument name, a numerical value, and an
optional set of labels.  Labels are key:value pairs associated with
events describing various dimensions or categories that describe thee
event.  The Metrics API supports applying explicit labels through the
API itself, while labels can also be applied to metric events
implicitly, through the current OpenTelemetry context and resources.

### Handles and LabelSets

Metric instruments support a _Handle_ interface.  Metric handles are a
pair consisting of an instrument and a specific set of pre-defined
labels, allowing for efficient repeated measurements.  The use of
pre-defined labels is so important for performance that we make it a
first-class concept in the API.

A `LabelSet` is an API object, returned by the SDK through
`Meter.DefineLabels(labels)`, that represents a set of "witnessed"
labels.  Applications cannot read the labels belonging to a `LabelSet`
object, they are simply a reference to a specific
`Meter.DefineLabels()` event.

Handles and LabelSets support different ways to achieve the same kind
of optimization.  Generally, there is a high cost associated with
computing a canonicalized form of the label set that can be used as a
map key, in order to look up a corresponding group entry for
aggregation.  Handles offer the most optimization potential, but
require the programmer to allocate and store one handle per metric.
LabelSets offer most of the optimization potential without managing
one handle per metric.

### Export pipeline

A metrics export pipeline consists of an SDK-provided `Meter`
implementation that processes metrics events (somehow), paired with a
metrics data exporter that sends the data (somehow).

### Required label keys

Metric instruments list an optional set of _required label keys_ to
support potential optimizations in the metric export pipeline.  There
is an important relationship between metric Instruments, their
required label keys, and LabelSets.  To support efficient
pre-aggregation in the metrics export pipeline, we must be able to
infer the value of the required label keys from the `LabelSet`, which
implies that they are not dependent on dynamic context. All this means
that the value of a required label key is explicitly unspecified
unless it is provided in the `LabelSet`, by definition, and not to be
taken from other context.

### Input methods

The API surface supports three ways to produce metrics events, after
obtaining a re-usable `LabelSet` via `Meter.DefineLabels(labels)`:

1. Through the Instrument directly.  Use `Counter.Add(LabelSet,
value)`, `Gauge.Set(LabelSet, value)`, or `Measure.Record(LabelSet,
value)` to produce an event (with no Handle).

2. Through an instrument Handle.  Use the
`Instrument.GetHandle(LabelSet)` API to construct a new handle that
implements the respective `Add(value)`, `Set(value)`, or
`Record(value)` method.

3. Through a `Meter.RecordBatch` call.  Use the batch API to enter
simultaneous measurements, where a measurement is a tuple consisting
of the `Instrument`, a `LabelSet` and the value for the appropriate
`Add()`, `Set()`, or `Record()` method.

For instrument `Monotonic` (Counter: default), `Monotonic` (Gauge:
option), and `NonNegative` options (Measure: default), certain inputs
are invalid.  These invalid inputs must not panic or throw unhandled
exceptions, however the SDK chooses to handle them.
