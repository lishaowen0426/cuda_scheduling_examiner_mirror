CUDA Scheduling Viewer
======================

About
-----

This project was intended to provide a tool for examining block-level
scheduling behavior and coscheduling performance on CUDA devices. The tool is
capable of running any benchmark which can be self-contained in a shared
library file exporting specific functions. Currently, this tool only runs under
Linux, and is unlikely to support other systems in the future.


Compilation
-----------

This tool can only be run on a computer with a CUDA-capable GPU and with CUDA
installed. The `nvcc` command must be available on your PATH. The tool has not
been tested with devices earlier than compute capability 3.0 or CUDA versions
earlier than 8.0. GCC version 4.9 or later is required.

To build, clone the repository, `cd` into it, and run `make`.

Usage
-----

The tool must be provided a JSON configuration file, which will contain
information about which benchmark libraries to run, how to run them, and what
parameters to provide. The file `configs/simple.json` has been provided as a
minimal example, running one instance of the `mandelbrot.so` benchmark. To run
it:

```bash
./bin/runner ./configs/simple.json
```

Additionally, the character `-` may be used in place of a config file name, in
which case the tool will attempt to read a JSON configuration object from
stdin. The file will be read completely before any benchmarks begin execution.

Some scripts have been included to visualize results. They require python,
numpy, and matplotlib. All such scripts are located in the scripts directory.
For example:

```bash
# Run all known configurations
find configs/*.json -exec ./bin/runner {} \;

# Visualize the scheduling timelines for each scenario
python scripts/view_timelines.py
```

Configuration Files
-------------------

The configuration files specify parameters passed to each benchmark along with
some global settings for the entire program.

The layout of each configuration file is as follows:

```
{
  "name": <String. Required. The name of this scenario.>,
  "max_iterations": <Number. Required. Default cap on the number of iterations
    for each benchmark. 0 = unlimited.>,
  "max_time": <Number. Required. Default cap on the number of number of seconds
    to run each benchmark. 0 = unlimited.>,
  "use_processes": <Boolean, defaulting to false. If this is true, each
    benchmark is run in a separate process. Normally, they run as threads.>
  "cuda_device": <Number. Required. The CUDA device to use for benchmarks to
    use.>,
  "base_result_directory": <String, defaulting to "./results". This is the
    directory into which individual JSON files from each benchmark will be
    written. It must already exist.>,
  "pin_cpus": <Boolean. Optional, defaults to false. If true, attempt to pin
    benchmarks to CPU cores, evenly distributed across cores. If true,
    individual benchmark cpu_core settings are ignored.>,
  "sync_every_iteration": <Boolean. Optional, defaults to false. If true,
    iterations of each benchmark start when all benchmarks have completed their
    previous iteration. By default, each benchmark only waits for its own
    previous iteration to complete.>,
  "benchmarks": [
    {
      "filename": <String. Required. The path to the benchmark file, relative
        to the current working directory.>,
      "log_name": <String. Optional. The filename of the JSON log for this
        particular benchmark. If not provided, this benchmark's log will be
        given a default name based on its filename, process and thread ID. If
        this doesn't start with '/', it will be relative to
        base_result_directory.>,
      "label:": <String. Optional. A label or name for this specific benchmark,
        to be copied to its output file.>,
      "thread_count": <Number. Required, but may be ignored. The number of CUDA
        threads this benchmark should use.>,
      "block_count": <Number. Required, but may be ignored. The number of CUDA
        blocks this benchmark should use.>,
      "data_size": <Number. Required, but may be ignored. The input size, in
        bytes, for the benchmark.>,
      "additional_info": <A JSON object of any format. Optional. This can be
        used to pass additional benchmark-specific configuration parameters.>,
      "max_iterations": <Number. Optional. If specified, overrides the default
        max_iterations for this benchmark alone. 0 = unlimited. If this is
        provided for any benchmark, then sync_every_iteration must be false.>,
      "max_time": <Number. Optional. If specified, overrides the default
        max_time for this benchmark alone. 0 = unlimited.>,
      "release_time": <Number. Optional. If set, this benchmark will sleep for
        the given number of seconds (between initialization and the start of
        the first iteration) before beginning execution.>,
      "cpu_core": <Number. Optional. If specified, and pin_cpus is false, the
        system will attempt to pin this benchmark onto the given CPU core.>
      "stream_priority": <Number. Optional. If specified, and is an integer in
        the range [-1,0], the stream will be created with priority. -1 is higher
        priority and 0 is lower.
    },
    {
      <more benchmarks can be listed here>
    }
  ]
}
```

Additionally, benchmark configurations support the insertion of comments via
the usage of "comment" keys, which will be ignored at runtime.


Automatic Benchmark Generation
------------------------------

The script located in `scripts/multikernel_generator.py` illustrates how
config generation can be scripted. To run a scenario automatically generated by
this script, run the following command (after running `make`):

```bash
python scripts/multikernel_generator.py | ./bin/runner -
```

VisionWorks Demos Benchmark
---------------------------

NVIDIA's VisionWorks demos are available to use as benchmarks. These demos are
real-world autonomous driving applications such as stereo matching, feature
tracking, the Hough transform, and motion stabilization. The source code of
these demos was modified to fit the benchmark interface. The modified source
code is located in `src/third_party/VisionWorks-1.6-Demos`.

To compile VisionWorks benchmarks, use `make visionworks`. The VisionWorks
library must be installed for this to complete.

The `scripts/visionworks_generator.py` script is provided to generate
configuration files for VisionWorks demos. Check out detailed help information
using `python scripts/visionworks_generator.py -h`. Here is an example running
one feature tracker instance:

```bash
python scripts/visionworks_generator.py --ft 1 | ./bin/runner -
```

To run VisionWorks case study, use:

```bash
./scripts/visionworks_scenarios_case_study.sh  # run the case study
python scripts/view_times_cdf.py -k "execute_times" # view CDF plots
python scripts/view_times_pdf.py # view PDF (probablity density function) plots,
# by default it only shows "x4" scenarios. This can be modified in the script
```

Output File Format
------------------

Each benchmark, when run, will generate a JSON log file at the location
specified in the configuration. If the benchmark did not complete successfully,
the JSON file may be in an invalid state. Times will be recorded as
floating-point numbers of seconds. The format of the log file is:

```
{
  "scenario_name": "<Scenario name>",
  "benchmark_name": "<Benchmark name>",
  "label": "<This benchmark's label, if given in the config>",
  "max_resident_threads": <The maximum number of threads that can be assigned
    to the GPU at a time (from all benchmarks in the scenario)>,
  "data_size": <Data size>,
  "release_time": <Requested release time in seconds>,
  "PID": <pid>,
  "TID": <The thread ID, if benchmarks were run as threads>,
  "times": [
    {},
    {
      "cpu_times": [
        <The CPU time before the copy_in function was called>,
        <The CPU time after the copy_out function returned>
      ],
      "copy_in_times": [
        <The CPU time before the copy_in function was called>,
        <The CPU time after the copy_in function returned>
      ],
      "execute_times": [
        <The CPU time when the execute function was called>,
        <The CPU time after the execute function returned>
      ],
      "copy_out_times": [
        <The CPU time when the copy_out function was called>,
        <The CPU time after the copy_out function returned>
      ]
    },
    {
      "kernel_name": <The name of this particular kernel. May be omitted.>,
      "block_count": <The number of blocks in this kernel invocation.>,
      "thread_count": <The number of threads per block in this invocation.>,
      "shared_memory": <The amount of shared memory used by this kernel.>,
      "cuda_launch_times": [<CPU time immediately before the kernel launch.>,
        <CPU time immediately after kernel launch returned.>,
        <CPU time immediately after cudaStreamSynchronize returned. This will
        be set to 0 if cudaStreamSynchronize isn't called for this kernel.>],
      "block_times": [<Start time>, <End time>, ...],
      "block_smids": [<Block 0 SMID>, <Block 1 SMID>, ...],
      "cpu_core": <The current CPU core being used>
    },
    ...
  ]
}
```

Notice that the first entry in the "times" array will be blank and should be
ignored. The times array will contain two types of objects: one will contain
CPU times and one type will apply to kernel times. An object containing CPU
times will contain a `"cpu_times"` key. A single CPU times object will
encompass all kernel times following it, up until another CPU times object.

Creating New Benchmarks
-----------------------

Each benchmark must be contained in a shared library and abide by the interface
specified in `src/library_interface.h`. In particular, the library must export
a `RegisterFunctions` function, which provides the addresses of further
functions to the calling program. Benchmarks, preferably, should never use
global state and instead use the `user_data` pointer returned by the
initialize function to track all state. Global state may function if only one
instance of each benchmark is run at a time, but this will never be a
limitation of the default benchmarks included in this project. All benchmarks
must use a user-created CUDA stream in order to avoid unnecessarily blocking
each other.

The most important piece of information that each benchmark provides is the
`TimingInformation` struct, filled in during the `copy_out` function of each
benchmark. This struct will contain a list of `KernelTimes` structs, one for
each kernel invocation called during `execute`. Each `KernelTimes` struct will
contain the kernel start and end times, individual block start and end times,
and a list of the SM IDs to which blocks were assigned. The benchmark is
responsible for ensuring that the buffers provided in the TimingInformation
struct remain valid at least until another benchmark function is called. They
will not be freed by the caller.

In general, the comments in `library_interface.h` provide an explanation for
the actions that every library-provided function is expected to carry out.
The existing libraries in `src/mandelbrot.cu` and `src/timer_spin.cu` provide
examples of working library implementations.

In addition to `library_interface.h`, `benchmark_library_funcions.h/cu` define
a library of utility functions that may be shared between benchmarks.

Benchmark libraries are invoked by the master process as follows:

 1. The shared library file is loaded using the `dlopen()` function, and the
    `RegisterFunctions` function is located using `dlysym()`.

 2. Depending on the configuration, either a new process or new thread will be
    created for each benchmark.

 3. In its own thread or process, the benchmark's `initialize` function will
    be called, which should allocate and initialize all of the local state
    necessary for one instance of the benchmark.

 4. When the benchmark begins running, a single iteration will consist of the
    benchmark's `copy_in`, `execute`, and `copy_out` functions being called, in
    that order.

 5. When enough time has elapsed or the maximum number of iterations has been
    reached, the benchmark's `cleanup` function will be called, to allow for
    the benchmark to clean up and free its local state.

Coding Style
------------

Even though CUDA supports C++, contributions to this project should use the C
programming language when possible. C or CUDA source code should adhere to the
parts of the [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)
that apply to the C language.

Scripts should remain in the `scripts/` directory and should be written in
python when possible. For now, there is no explicit style guide for python
scripts apart from trying to maintain a consistent style within each file.
