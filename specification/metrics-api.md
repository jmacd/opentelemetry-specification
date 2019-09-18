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

The OpenTelemetry API that provides the metrics SDK is named `Meter`,
and whereas the `Meter` ultimately determines the treatment of metric
events at runtime, this specification can only define their semantics
and standard interpretation.  The three semantic kinds of instrument
are distinguished primarily by the verb they support, which implies
their meaning.  

### Counter

Counters support `Add(value)`, signifying in the code and to the SDK
that a sum or a rate is of primary interest.  Counters are defined as
monotonic by default, meaning that negative values are invalid.
Monotonic counters are typically used because they can automatically
be interpreted as a rate.

As an option, counters can be declared as `NonMonotonic`, in which
case they support positive and negative increments.  Non-monotonic
counters are useful to report changes in an accounting scheme, such as
a number of bytes allocated.

### Gauge

Gauges support `Set(value)`, signifying in the code and to the SDK
that events modify a current value.  Gauges are defined as
non-monotonic by default, meaning that any value (positive or
negative) is allowed.

As an option, gauges can be declared as `Monotonic`, in which case
successive values must rise monotonically.  Monotonic gauges are
useful in reporting computed sums, allowing an application to compute
a current value and report it, without remembering the last-reported
value. 

A special case of gauge is supported, called an `Observer` metric
instrument, which is semantically equivalent to a gauge but uses a
callback to report the current value.  Observer instruments are
defined by a callback, instead of supporting `Set()`, but the
semantics are the same.  The only difference between `Observer` and
ordinary gauges is that their events do not have an associated
OpenTelemetry context.  Observer instruments are non-monotonic by
default and monotonic as an option, like ordinary gauges.

### Measure

Measures support `Record(value)`, signifying in the code and to the
SDK that events report individual measurements.  Measures are defined
as `NonNegative` by default, meaning that negative values are invalid.
Non-negative measures are typically used to record absolute values
such as durations and sizes.

As an option, measures can be declared as `Signed` to indicate support
for positive and negative values.  TODO: term for `Signed` ok?

### Discussion

Value types: Histogram, Summary metrics. XXX

## Detailed Description

### Structure

Metric instruments are named.  Regardless of the instrument kind,
metric events include the instrument name, a numerical value, and an
optional set of labels.  Labels are key:value pairs associated with
events describing various dimensions or categories that describe the
event.  The Metrics API supports applying explicit labels through the
API itself, while labels can also be applied to metric events
implicitly, through context and resources, as a benefit of
OpenTelemetry.

### Handles

Metric instruments support a _Handle_ interface.  Metric handles are a
pair consisting of an instrument and a specific set of pre-defined
labels, allowing for efficient repeated measurements.  The use of
pre-defined labels is so important for performance that we make it a
first-class concept in the API.  A `LabelSet` is an API object,
returned by the SDK through `Meter.DefineLabels`, that represents a
set of recorded labels.  Applications cannot read the labels belonging
to a `LabelSet` object, they are simply a reference to a specific
`Meter.DefineLabels` event.

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