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

Metric instruments list an optional set of _required label keys_ to
support an important potential optimization.
