# Metrics API

The _Metrics API_ supports reporting diagnostic measurements using
three basic kinds of instrument, commonly known as Counters, Gauges,
and Measures.  Instruments are code-level objects, allocated for and
handled by programmers, used to gain visibility into operational
metrics about the application, its components, and the system as a
whole.

Monitoring and alerting are the common use-case for the data provided
by metric instruments, after various collection and aggregation
strategies are applied to the data.  We find there are many other uses
for the _metric events_ that stream into these instruments, through
their respective `Add()`, `Set()`, and `Record()` methods.  We imagine
metric data being aggregated and recorded as events in tracing and
logging systems too, and for this reason OpenTelemetry requires a
separation of the API from the SDK.

The OpenTelemetry API that provides the metrics SDK is named `Meter`.
Whereas the `Meter` ultimately determines how metrics events will be
handled at runtime, this specification can only define their
semantics.  The three semantic kinds of instrument are distinguished
primarily by the verb they use.  Counters use `Add()`.  Gauges use
`Set()`.  Measures use `Record()`.

Metric instruments are named.  Regardless of the instrument kind,
metric events include the instrument name, a numerical value, and an
optional set of labels.  Labels are key:value pairs associated with
events describing various dimensions or categories that describe the
event.  The Metrics API supports applying explicit labels in several
ways, an area where performance concerns commonly arise.  Programmers
apply labels to metric events implicitly through context and resources
as well.

Metric instruments support _Handles_, which are a coupling between a
specified instrument and a set of pre-defined labels allowing for
re-use.  The use of pre-defined labels is so important for performance
that we make it a first-class concept in the API.  A `LabelSet` is an
API object, returned by the SDK through `Meter.DefineLabels`, that
represents a set of recorded labels.  Applications cannot read the
labels belonging to a `LabelSet` object, they are simply a reference
to a specific `Meter.DefineLabels` event.

A metrics export pipeline consists of an SDK-provided `Meter`
implementation that processes metrics events (somehow), paired with a
metrics data exporter that sends the data (somehow).

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

The API surface supports three ways to produce metrics events:

1. Through an Instrument Handles.  Use
`Instrument.GetHandle(LabelSet)` API constructs a new handle, which
implements the respective `Add(value)`, `Set(value)`, or
`Record(value)` method.

2. Through the Instrument directly.  Use `Counter.Add(LabelSet,
value)`, `Gauge.Set(LabelSet, value)`, or `Measure.Record(LabelSet,
value)` to produce an event without a handle.

3. Through a `Meter.RecordBatch` call.  Use the batch API to enter
simultaneous measurements, where a measurement is a tuple consisting
of the `Instrument`, a `LabelSet` and the value for the appropriate
`Add()`, `Set()`, or `Record()` method.

Monotonic, Non-Monotonic, Non-Negative, and Rates

...

