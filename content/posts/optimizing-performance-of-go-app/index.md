+++
title = "How to benchmark performance of Go serializers"
description = "Learn how to benchmark the performance of your Go code to make informed decisions"
authors = ["Victor Lyuboslavsky"]
image = "race-cars-headline.png"
date = 2024-08-28
categories = ["Software Development"]
tags = ["Golang", "Performance"]
draft = false
+++

- [Creating a Go benchmark](#creating-go-benchmark)
- [Running Go benchmarks](#running-go-benchmarks)

## What is benchmarking?

Performance optimization is a critical part of software development. Once your application has been released and is
being used by real users, you may need to optimize its performance. One way to do this is to benchmark your code to
identify bottlenecks and improve its performance. Benchmarking provides you with data to make informed decisions about
what parts of your code can be sped up and by how much.

Benchmarking is the process of measuring your code's performance. It involves running your code multiple times and
measuring how long it takes to execute. By running your code multiple times, you can get an average execution time,
which is more reliable than a one-off report.

## Identifying the bottlenecks

In our application, we deserialize and process large amounts of JSON data once every hour. We noticed that this process
was taking a long time for some of our users. First, we used
[Go pprof](https://github.com/google/pprof/blob/main/doc/README.md) to enable profiling and generated a flame graph to
identify the bottlenecks in our code.

{{< figure src="go-pprof-flame-graph.png" title="Go pprof flame graph" alt="Flame graph showing the execution time of different parts of the Go app" >}}

The flame graph showed us that the JSON decoding process took the most time. We benchmarked different serialization
libraries to find the fastest one for our use case.

## Creating a Go benchmark {#creating-go-benchmark}

In Go, you can write benchmarks using the built-in testing package. Benchmarks are written similarly to unit tests but
with the **Benchmark** prefix instead of the **Test** prefix.

Before creating and running the benchmark, we generated 1000 test JSON files in the `testdata` directory.

To benchmark JSON decoding, we created the following benchmark.

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
    "testing"
)

const files = 1000
const itemsPerFile = 1000

func BenchmarkJSONImport(b *testing.B) {
    for i := 0; i < b.N; i++ {

       // Read in the file (do not time)
       b.StopTimer()
       fileNumber := i % files
       data, err := os.ReadFile(fmt.Sprintf("testdata/sample_%d.json", fileNumber))
       if err != nil {
          b.Fatal(err)
       }
       b.StartTimer()

       var samples []Sample
       dec := json.NewDecoder(bytes.NewReader(data))
       if err := dec.Decode(&samples); err != nil {
          b.Fatal(err)
       }
       if len(samples) != itemsPerFile {
          b.Fatalf("expected %d samples, got %d", itemsPerFile, len(samples))
       }
    }
}
```

Starting the function name with `Benchmark` indicates to `go test` that this is a benchmark.

The testing package adjusts the number of iterations through the `for i := 0; i < b.N; i++` loop until the function
lasts long enough to be timed reliably.

The `b.StopTimer()` and `b.StartTimer()` calls exclude part of the code from the benchmark.

## Running Go benchmarks {#running-go-benchmarks}

To run all benchmarks, add `-bench=.` flag to `go test`:

```bash
go test -bench=.
```

The result will look like this:

```
goos: darwin
goarch: arm64
pkg: serializer
cpu: Apple M2 Pro
BenchmarkJSONImport-12           357      3324997 ns/op
PASS
ok      serializer 1.868s
```

It tells us that unmarshalling a single file with `json.Decode` takes an average of 3.3 milliseconds. The benchmark ran
the loop 357 times.

## Benchmarking encoding/gob

Next, we will benchmark the built-in [encoding/gob](https://pkg.go.dev/encoding/gob) library.

```go
func BenchmarkGobImport(b *testing.B) {
    for i := 0; i < b.N; i++ {

       // Read in the file (do not time)
       b.StopTimer()
       fileNumber := i % files
       data, err := os.ReadFile(fmt.Sprintf("testdata/sample_%d.bin", fileNumber))
       if err != nil {
          b.Fatal(err)
       }
       b.StartTimer()

       // decode gob
       var samples []Sample
       dec := gob.NewDecoder(bytes.NewReader(data))
       if err := dec.Decode(&samples); err != nil {
          b.Fatal(err)
       }
       if len(samples) != itemsPerFile {
          b.Fatalf("expected %d samples, got %d", itemsPerFile, len(samples))
       }
    }
}
```

Running the two benchmarks gives us:

```
BenchmarkJSONImport-12           360      3279579 ns/op
BenchmarkGobImport-12           2262       475469 ns/op
```

The benchmark data shows that decoding with `encoding/gob` takes almost 7 times faster than using `encoding/json`. This
gives sufficient data to present to our management and argue for switching from JSON. In addition, we can benchmark
other serialization libraries to see if any of them are even faster.

For additional data, we included reading the file in our benchmark numbers for a complete picture of the expected
speedup:

```
BenchmarkJSONImport-12               360      3374064 ns/op
BenchmarkGobImport-12               2710       481935 ns/op
BenchmarkJSONImportFile-12           306      3746758 ns/op
BenchmarkGobImportFile-12           2176       554254 ns/op
```

## Go benchmark code on GitHub

The complete code is available on GitHub at: https://github.com/getvictor/go-benchmark-serializers

## Further reading

- Recently, we wrote about [accurately measuring Go test execution time](../go-test-execution-time).
- Also, see our previous article on [creating fuzz tests in Go](../fuzz-testing-with-go).

## Watch how to benchmark Go serializers

{{< youtube c0drQ2JUYmo >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
