# Metrics APIs Design

## Overview

This document is discussing the .NET Metrics APIs design which implements the [OpenTelemetry specification](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/metrics/new_api.md).

The OpenTelemetry Metrics APIs support reporting measurements about the execution of a computer program at run time. The Metrics APIs are designed explicitly for processing raw measurements, generally with the intent to produce continuous summaries of those measurements, efficiently and simultaneously.

The OpenTelemetry architecture is splitting the measurement reporting part to a component called `APIs` and that is the part we are discussing here and proposing APIs for. The other part is called `SDK` which will monitor, aggregate the Metrics measurements, and export the aggregated measurement to some backend. The SDK part will be implemented in the [OpenTelemetry .NET repo](https://github.com/open-telemetry/opentelemetry-dotnet) and will not be part of .NET. The `SDK` will depend on the APIs we are going to expose in .NET.

The proposed APIs will be used by the application or library authors to record metrics measurements. The APIs also include a listener which will allow listening to the recorded measurements events which can be used by the OpenTelemetry SDK (or other potential consumers) to aggregate the measurements and export them.

The proposed APIs are intended to be supported on Full Framework 4.6 and up, .NET Core supported versions, and .NET 5.0 and up. The proposed APIs will be part of the System.Diagnostics.DiagnosticSource package.

```MD040
Naming in this proposal is picked up from the OpenTelemetry specification to avoid having .NET APIs
deviating from the standard even if the standard names are not the best option for the .NET ecosystem.
```

## Metric Terminologies

### Instrument

The Instrument is the type that will be used by the app and library authors to report measurements (e.g. Counter, Gauge...etc.). The instrument will have a name that should be validated as described in the OpenTelemetry specs. The Instrument can optionally have a description and unit of measurements. Instruments will be created with a numerical type parameter (e.g. `Counter<int>`). The Instrument is going to support all CLS-compliant numerical types which are supported on the Full Framework (Byte, Int16, Int32, Int64, Single, Double, and Decimal).

There are two types of instruments:

- The first type we'll call it just `Instrument` for simplicity. These instruments are called inside a request, meaning they have an associated distributed Context (with Span, Baggage, etc.). OpenTelemetry specs call this type of instrument a synchronous Instrument but we are trying to avoid confusion with the async feature of the .NET. The proposal here proposes three instrument classes of that type: `Counter`, `Gauge` and `Histogram`.
- The second type is called `ObservableInstrument` which reports measurements by a callback, once per collection interval, and lacks Context. OpenTelemetry specs call this type of instrument an asynchronous Instrument but we are trying to avoid confusion with the async feature of the .NET. The proposal here is proposing three instrument classes of that type: `ObservableCounter`, `ObservableGauge`, and `ObservableUpDownCounter`.

### Meter

Meter is the factory type responsible for creating Instruments. Meter will have a name and optional version strings.

### MeterListener

The listener is the type that allows listening to the measurements reported by instruments (e.g. Counter, ObservableCounter, etc.). The listener is an important class that will be used by OpenTelemetry to implement the Metrics SDK.

### Tags

Tag is the term used to refer to a key-value attribute associated with a metric event. Each tag categorizes the metric event, allowing events to be filtered and grouped for analysis.
Current OpenTelemetry specs call it Attributes. We have used the tags name in the tracing APIs and we'll stick with this name for the sake of consistency with tracing.

## APIs Proposal

### Meter Class

```csharp
namespace System.Diagnostics.Metrics
{
    public class Meter : IDisposable
    {

        /// <summary>
        /// The constructor allows creating the Meter class with the name and optionally the version.
        /// The name should be validated as described by the OpenTelemetry specs.
        /// </summary>
        public Meter(string name)  { throw null;  }
        public Meter(string name, string? version) { throw null; }

        /// <summary>
        /// Getter properties to retrieve the Meter name and version
        /// </summary>
        public string Name { get; }
        public string? Version { get; }

        /// <summary>
        /// Factory Methods to create Counter and Histogram instruments.
        /// </summary>
        public Counter<T> CreateCounter<T>(
                            string name,
                            string? description = null,
                            string? unit = null) where T : unmanaged { throw null; }

        public Histogram<T> CreateHistogram<T>(
                            string name,
                            string? description = null,
                            string? unit = null) where T : unmanaged { throw null; }

        /// <summary>
        /// Factory Methods to create an observable Counter instrument.
        /// </summary>

        public ObservableCounter<T> CreateObservableCounter<T>(
                            string name,
                            Func<T> observeValue,
                            string? description = null,
                            string? unit = null) where T : unmanaged { throw null; }

        public ObservableCounter<T> CreateObservableCounter<T>(
                            string name,
                            Func<Measurement<T>> observeValue,
                            string? description = null,
                            string? unit = null) where T : unmanaged { throw null; }

        public ObservableCounter<T> CreateObservableCounter<T>(
                            string name,
                            Func<IEnumerable<Measurement<T>>> observeValues,
                            string? description = null,
                            string? unit = null) where T : unmanaged { throw null; }

        /// <summary>
        /// Factory Methods to create observable gauge instrument.
        /// </summary>
        public ObservableGauge<T> CreateObservableGauge<T>(
                            string name,
                            Func<T> observeValue,
                            string? description = null,
                            string? unit = null) where T : unmanaged { throw null; }

        public ObservableGauge<T> CreateObservableGauge<T>(
                            string name,
                            Func<Measurement<T>> observeValue,
                            string? description = null,
                            string? unit = null) where T : unmanaged { throw null; }

        public ObservableGauge<T> CreateObservableGauge<T>(
                            string name,
                            Func<IEnumerable<Measurement<T>>> observeValues,
                            string? description = null,
                            string? unit = null) where T : unmanaged { throw null; }

        /// <summary>
        /// Factory Methods to create observable UpDownCounter instrument.
        /// </summary>
        public ObservableUpDownCounter<T> CreateObservableUpDownCounter<T>(
                            string name,
                            Func<T> observeValue,
                            string? description = null,
                            string? unit = null) where T : unmanaged { throw null; }

        public ObservableUpDownCounter<T> CreateObservableUpDownCounter<T>(
                            string name,
                            Func<Measurement<T>> observeValue,
                            string? description = null,
                            string? unit = null) where T : unmanaged { throw null; }

        public ObservableUpDownCounter<T> CreateObservableUpDownCounter<T>(
                            string name,
                            Func<IEnumerable<Measurement<T>>> observeValues,
                            string? description = null,
                            string? unit = null) where T : unmanaged { throw null; }

        public void Dispose() { throw null; }
    }
}
```

### Instrument Base Class

```csharp
namespace System.Diagnostics.Metrics
{
    /// <summary>
    /// Is the base class which contains all common properties between different types of instruments.
    /// It contains the protected constructor and the Publish method allows activating the instrument
    /// to start recording measurements.
    /// </summary>

    public abstract class Instrument
    {
        /// <summary>
        /// Protected constructor to initialize the common instrument properties.
        /// </summary>
        protected Instrument(Meter meter, string name, string? description, string? unit) { throw null; }

        /// <summary>
        /// Publish is to allow activating the instrument to start recording measurements and to allow
        /// listeners to start listening to such measurements.
        /// </summary>
        protected void Publish() { throw null; }

        /// <summary>
        /// Getters to retrieve the properties that the instrument is created with.
        /// </summary>
        public Meter Meter { get; }
        public string Name { get; }
        public string? Description { get; }
        public string? Unit { get; }

        /// <summary>
        /// A property tells if a listener is listening to this instrument measurement recording.
        /// </summary>
        public bool Enabled => throw null;

        /// <summary>
        /// A property tells if the instrument is a regular instrument or an observable instrument.
        /// </summary>
        public virtual bool IsObservable => throw null;
    }
}
```

### Non-observable Instrument Base Class

```csharp
namespace System.Diagnostics.Metrics
{
    /// <summary>
    /// Instrument<T> is the base class from which all non-observable instruments will inherit from.
    /// Mainly It'll support the CLS compliant numerical types
    /// </summary>
    public abstract class Instrument<T> : Instrument where T : unmanaged
    {
        /// <summary>
        /// Protected constructor to create the instrument with the common properties.
        /// </summary>
        protected Instrument(Meter meter, string name, string? description, string? unit) :
                        base(meter, name, description, unit) { throw null; }

        /// <summary>
        /// Record measurement overloads allowing passing different numbers of tags.
        /// </summary>

        protected void RecordMeasurement(T measurement) { throw null; }

        protected void RecordMeasurement(
                            T measurement,
                            KeyValuePair<string, object?> tag) { throw null; }

        protected void RecordMeasurement(
                            T measurement,
                            KeyValuePair<string, object?> tag1,
                            KeyValuePair<string, object?> tag2) { throw null; }

        protected void RecordMeasurement(
                            T measurement,
                            KeyValuePair<string, object?> tag1,
                            KeyValuePair<string, object?> tag2,
                            KeyValuePair<string, object?> tag3) { throw null; }

        protected void RecordMeasurement(
                            T measurement,
                            ReadOnlySpan<KeyValuePair<string, object?>> tags) { throw null; }
    }
}
```

### Observable Instrument Base class

```csharp
namespace System.Diagnostics.Metrics
{
    /// <summary>
    /// ObservableInstrument<T> is the base class from which all observable instruments will inherit from.
    /// It will only support the CLS compliant numerical types
    /// </summary>
    public abstract class ObservableInstrument<T> : Instrument where T : unmanaged
    {
        /// <summary>
        /// Protected constructor to create the instrument with the common properties.
        /// </summary>
        protected ObservableInstrument(
                    Meter meter,
                    string name,
                    string? description,
                    string? unit) : base(meter, name, description, unit) { throw null; }

        /// <summary>
        /// Observe() fetches the current measurements being tracked by this instrument.
        /// </summary>
        protected abstract IEnumerable<Measurement<T>> Observe();

        public override bool IsObservable => throw null;
    }
}
```

### Measurement class

```csharp

namespace System.Diagnostics.Metrics
{
    /// <summary>
    /// A measurement stores one observed value and its associated tags. This type is used by Observable instruments' Observe() method when reporting current measurements.
    /// with the associated tags.
    /// </summary>
    public struct Measurement<T> where T : unmanaged
    {
        /// <summary>
        /// Construct the Measurement using the value and the list of tags.
        /// We'll always copy the input list as this is not perf hot path.
        /// </summary>
        public Measurement(T value, IEnumerable<KeyValuePair<string, object?>> tags) { throw null; }
        public Measurement(T value, params KeyValuePair<string, object?>[] tags) { throw null; }
        public Measurement(T value, ReadOnlySpan<KeyValuePair<string, object?>> tags) { throw null; }

        public ReadOnlySpan<KeyValuePair<string, object?>> Tags { get { throw null;  } }
        public T Value { get; }
    }
}
```

### Non-observable Instruments

```csharp
namespace System.Diagnostics.Metrics
{
    /// <summary>
    /// The counter is a non-observable Instrument that supports non-negative increments.
    /// e.g. Number of completed requests.
    /// </summary>
    public sealed class Counter<T> : Instrument<T> where T : unmanaged
    {
        public void Add(T measurement) { throw null; }
        public void Add(T measurement,
                            KeyValuePair<string, object?> tag) { throw null; }
        public void Add(T measurement,
                            KeyValuePair<string, object?> tag1,
                            KeyValuePair<string, object?> tag2) { throw null; }
        public void Add(T measurement,
                            KeyValuePair<string, object?> tag1,
                            KeyValuePair<string, object?> tag2,
                            KeyValuePair<string, object?> tag3) { throw null; }
        public void Add(T measurement,
                            ReadOnlySpan<KeyValuePair<string, object?>> tags) { throw null; }
        public void Add(T measurement,
                            params KeyValuePair<string, object?>[] tags) { throw null; }
    }

    /// <summary>
    /// The histogram is a non-observable Instrument that can be used to report arbitrary values
    /// that are likely to be statistically meaningful. It is intended for statistics such
    /// e.g. the request duration.
    /// </summary>
    public sealed class Histogram<T> : Instrument<T> where T : unmanaged
    {
        public void Record(T measurement) { throw null; }
        public void Record(
                        T measurement,
                        KeyValuePair<string, object?> tag) { throw null; }
        public void Record(
                        T measurement,
                        KeyValuePair<string, object?> tag1,
                        KeyValuePair<string, object?> tag2) { throw null; }
        public void Record(
                        T measurement,
                        KeyValuePair<string, object?> tag1,
                        KeyValuePair<string, object?> tag2,
                        KeyValuePair<string, object?> tag3) { throw null; }
        public void Record(T measurement,
                            ReadOnlySpan<KeyValuePair<string, object?>> tags) { throw null; }
        public void Record(
                        T measurement,
                        params KeyValuePair<string, object?>[] tags) { throw null; }
    }
}

```

### Observable Instruments

```csharp
namespace System.Diagnostics.Metrics
{
    /// <summary>
    /// ObservableCounter is an observable Instrument that reports monotonically increasing value(s)
    /// when the instrument is being observed.
    /// e.g. CPU time (for different processes, threads, user mode or kernel mode).
    /// </summary>
    public sealed class ObservableCounter<T> : ObservableInstrument<T> where T : unmanaged
    {
        protected override IEnumerable<Measurement<T>> Observe() { throw null; }
    }

    /// <summary>
    /// ObservableUpDownCounter is an observable Instrument that reports additive value(s)
    /// when the instrument is being observed.
    /// e.g. the process heap size
    /// </summary>
    public sealed class ObservableUpDownCounter<T> : ObservableInstrument<T> where T : unmanaged
    {
        protected override IEnumerable<Measurement<T>> Observe() { throw null; }
    }

    /// <summary>
    /// ObservableGauge is an observable Instrument that reports non-additive value(s)
    /// when the instrument is being observed.
    /// e.g. the current room temperature
    /// </summary>
    public sealed class ObservableGauge<T> : ObservableInstrument<T> where T : unmanaged
    {
        protected override IEnumerable<Measurement<T>> Observe() { throw null; }
    }
}
```

### MeterListener

```csharp
namespace System.Diagnostics.Metrics
{
    /// <summary>
    /// A delegate to represent the callbacks signatures used in the listener.
    /// </summary>
    public delegate void MeasurementCallback<T>(
                            Instrument instrument,
                            T measurement,
                            ReadOnlySpan<KeyValuePair<string, object?>> tags,
                            object? state);


    /// <summary>
    /// The listener class can be used to listen to observable and non-observable instrument
    /// recorded measurements.
    /// </summary>
    public sealed class MeterListener : IDisposable
    {
        /// <summary>
        /// Simple constructor
        /// </summary>
        public MeterListener() { throw null;  }

        /// <summary>
        /// Callbacks to get notification when an instrument is published
        /// </summary>
        public Action<Instrument, MeterListener>? InstrumentPublished { get; set; }

        /// <summary>
        /// Callbacks to get notification when stopping the measurement on some instrument
        /// this can happen when the Meter or the Listener is disposed of. Or calling Stop()
        /// on the listener.
        /// </summary>
        public Action<Instrument, object?>? MeasurementsCompleted { get; set; }

        /// <summary>
        /// Start listening to a specific instrument measurement recording.
        /// </summary>
        public void EnableMeasurementEvents(Instrument instrument, object? state = null) { throw null; }

        /// <summary>
        /// Stop listening to a specific instrument measurement recording.
        /// returns the associated state.
        /// </summary>
        public object? DisableMeasurementEvents(Instrument instrument) { throw null; }

        /// <summary>
        /// Set a callback for a specific numeric type to get the measurement recording notification
        /// from all instruments which enabled listened to and was created with the same specified
        /// numeric type. If a measurement of type T is recorded and a callback of type T is registered, that callback is used. If there is no callback for type T but there is a callback for type object, the measured value is boxed and reported via the object typed callback. If there is neither type T callback nor object callback then the measurement will not be reported.
        /// </summary>
        public void SetMeasurementEventCallback<T>(MeasurementCallback<T>? measurementCallback) { throw null; }

        public void Start() { throw null; }
        public void Stop() { throw null; }

        /// <summary>
        /// Call all Observable instruments to get the recorded measurements reported to the
        /// callbacks enabled by SetMeasurementEventCallback<T>
        /// </summary>
        public void RecordObservableInstruments() { throw null; }

        public void Dispose() { throw null; }
    }
}
```

### Library Measurement Recording Example

```csharp
    Meter meter = new Meter("io.opentelemetry.contrib.mongodb", "v1.0");
    Counter<int> counter = meter.CreateCounter<int>("Requests");
    counter.Add(1);
    counter.Add(1, KeyValuePair.Create<string, object>("request", "read"));
```

### Listening Example

```csharp
    InstrumentListener listener = new InstrumentListener();
    listener.InstrumentPublished = (instrument, meterListener) =>
    {
        if (instrument.Name == "Requests" && instrument.Meter.Name == "io.opentelemetry.contrib.mongodb")
        {
            meterListener.EnableMeasurementEvents(instrument, null);
        }
    };
    listener.SetMeasurementEventCallback<int>((instrument, measurement, tags, state) =>
    {
        Console.WriteLine($"Instrument: {instrument.Name} has recorded the measurement {measurement}");
    });
    listener.Start();
```
