# `linkerd2-proxy` Benchmarking and Profiling

This directory contains a set of scripts and configurations for benchmarking and
profiling the Linkerd 2 proxy, using the benchmarking tools Fortio and iPerf.
Additionally, it contains scripts for profiling the proxy using these benchmark
configurations, as well as measuring latency.

## Requirements

All the benchmarks are run in Docker containers using `docker-compose`. For
most of the scripts in this directory, only `docker-compose` v3.7+ is required.

The `profiling-perf` script requires `perf` to be installed on the host, and is
only guaranteed to work on Linux (it may also work in Docker for Mac's Linux VM,
but your mileage may vary).

## Running Tests

This directory contains three test scripts:

- `benchmark.sh` performs a benchmark of the proxy, recording accurate latency
  information.
- `profiling-perf.sh` performs CPU profiling with `perf`, outputting a
  flamegraph of the proxy's CPU usage.
- `profiling-heap.sh` performs heap profiling using `memory_profiler`,
  outputting a flamegraph of heap allocations and a `.dat` file that may be
  further analyzed using `heaptrack` or `memory-profiler-cli`.

The profiling scripts perform the same _test_ as `benchmark.sh`, but the proxy
is configured to collect additional profiling information. Due to the overhead
of instrumenting the proxy to profile allocations or CPU usage, the latencies
recorded during profiling are probably **not** representative of proxy
performance "in the wild".

Whenever the proxy under test has changed (e.g. when making code changes, or
checking out a different branch to compare), it's necessary to rebuild the test
proxy Docker image. To do this, run the following command in the `profiling`
directory:

```console
$ docker-compose build proxy
```


### Customizing Parameters

The scripts in this directory take the following environment variables to
determine the test parameters.
When not provided they default to the values as listed here.

- `TCP="1"`: Enable/disable TCP benchmark.
- `HTTP="1"`: Enable/disable HTTP benchmark.
- `GRPC="1"`: Enable/disable gRPC benchmark.
- `ITERATIONS="1"`: The number of times a single HTTP/gRPC benchmark run is
  repeated to observe the maximal tail latency.
- `DURATION="10s"`: Execution time for a single HTTP/gRPC benchmark run.
- `CONNECTIONS="4"`: Number of concurrent TCP connections for HTTP/gRPC.
- `GRPC_STREAMS="4"`: Number of HTTP/2 streams in a connection.
- `HTTP_RPS="4000 7000"`: Different target HTTP req/s numbers as space-separated
  list. It may not be achieved if too high. Please see the actual req/s in the
  log output.
- `GRPC_RPS="4000 6000"`: As above but for gRPC.
- `REQ_BODY_LEN="10 200"`: Length of the request body payload in byte as
  space-separated list.
- `HIDE="1"`: Hide/show output of every command. The output of each benchmark
  utility is stored to log files in any case.

For example:

```console
$ ITERATIONS=2 DURATION=2s CONNECTIONS=2 GRPC_STREAMS=2 HTTP_RPS="100" GRPC_RPS="100 1000" REQ_BODY_LEN="100 8000" ./benchmark.sh
```

### Test Output

All test run output is written to the `target/profile/${TIMESTAMP}/` directory
in the root of the `linkerd2-proxy` repository. This includes the following:

- a summary CSV describing the observed 99th percentile latencies (or GBit/s for
  TCP)
- JSON files containing the raw data output by Fortio
- flamegraphs of the heap or CPU profile collected during the test, if running
  the profiling scripts
- `heaptrack.dat` files that can be analyzed using Heaptrack, when running the
  `profiling-heap.sh` script
- `.folded` perf output that may be used as input to generate additional
  flamegraphs, when running the `profiling-perf.sh` script

To remove all data generated by profiling, run:

```console
$ make clean-profile
```

from the root of the repository.


### Comparing the summary file of two branches

The `plot.sh` script generates graphs to compare the summary CSVs output by
different test runs. For example:

```console
$ cd profiling/
$ ./benchmark.sh ; git checkout main && ./benchmark.sh
$ ./plot.sh \
    ../target/profile/2019Jun19_15h13m12s/summary.txt \
    ../target/profile/.2019Jun19_15h34m26s/summary.txt \
    mybranch-vs-main-
$ eog mybranch-vs-main-*png
```

Consider using `./plot.sh --logy ...` to switch the Y-axis to a logarithmic
scale, if the difference between the values is too high to see the low values.

Another option is to quickly compare in textual form:

```console
$ git diff --no-index --word-diff \
    ../target/profile/2019Jun19_15h13m12s/summary.txt \
    ../target/profile/.2019Jun19_15h34m26s/summary.txt
```
