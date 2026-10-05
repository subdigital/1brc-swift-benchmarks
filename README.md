# Billion Row Challenge Benchmarks

This repo contains the Instruments traces and benchmark results
for the [Billion Row Challenge in Swift](https://github.com/.

## Mise Tasks

* `mise r go` - Build in release mode and run the program against `INPUT_FILE`
* `mise r verify` - Build in release mode and run the program, validate the output against `output.txt` to ensure correctness at each step
* `mise r benchmark` - Run hyperfine on the release binary. Save output to `benchmarks/{tag}.json`
* `mise r profile` - Run Instruments (Time Profiler) on the release binary. Append the run to the trace file.

These all assume a sibling `1brc` directory checked out to the tag that you want to build/profile.

## Benchmarks

The benchmarks were all run on a M4 Max Mac Studio with 64GB Memory.

