Split timeseries by data types in Metrics Protobuf definitions

Previously we had Point message which was a value of oneof the data types. This is
unnecessary flexibility because points in the same timeseries cannot be of different
data type. This also costed performance.

Now we have separate timeseries message definitions for each data type and the
timeseries used is defined by the oneof entry in the Metric message.

This change is stacked on top of https://github.com/open-telemetry/opentelemetry-proto/pull/33

Simple benchmark in Go demonstrates the following improvement of encoding and decoding
compared to the baseline state:

```
===== Encoded sizes
Encoding                       Uncompressed  Improved        Compressed  Improved
Baseline/MetricOne              24200 bytes  [1.000], gziped 1804 bytes  [1.000]
Proposed/MetricOne              19400 bytes  [1.247], gziped 1626 bytes  [1.109]

Encoding                       Uncompressed  Improved        Compressed  Improved
Baseline/MetricSeries           56022 bytes  [1.000], gziped 6655 bytes  [1.000]
Proposed/MetricSeries           43415 bytes  [1.290], gziped 6422 bytes  [1.036]

goos: darwin
goarch: amd64
pkg: github.com/tigrannajaryan/exp-otelproto/encodings
BenchmarkEncode/Baseline/MetricOne-8         	      27	 207923054 ns/op
BenchmarkEncode/Proposed/MetricOne-8         	      44	 133984867 ns/op

BenchmarkEncode/Baseline/MetricSeries-8      	       8	 649581262 ns/op
BenchmarkEncode/Proposed/MetricSeries-8      	      18	 324559562 ns/op

BenchmarkDecode/Baseline/MetricOne-8         	      15	 379468217 ns/op	186296043 B/op	 5274000 allocs/op
BenchmarkDecode/Proposed/MetricOne-8         	      21	 278470120 ns/op	155896034 B/op	 4474000 allocs/op

BenchmarkDecode/Baseline/MetricSeries-8      	       5	1041719362 ns/op	455096051 B/op	12174000 allocs/op
BenchmarkDecode/Proposed/MetricSeries-8      	       9	 603392754 ns/op	338296035 B/op	 8574000 allocs/op
```

It is 30-50% faster and is 20-25% smaller on the wire and in memory.

Benchmarks encode and decode 500 batches of 2 metrics: one int64 Gauge with 5 time series
and one Histogram of doubles with 1 time series and single bucket. Each time series for
both metrics contains either 1 data point (MetricOne) or 5 data points (MetricSeries).
Both metrics have 2 labels.

Benchmark source code is available at:
https://github.com/tigrannajaryan/exp-otelproto/blob/master/encodings/encoding_test.go
