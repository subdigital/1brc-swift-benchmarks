# Performance Notes

## `00-naive` → Raw-buffer parser

### Benchmark result

| Version | Mean | Standard deviation |
|---|---:|---:|
| `00-naive` | 242.78 s | 7.18 s |
| Current raw-buffer parser | 132.42 s | 0.92 s |

The raw-buffer parser is:

- **110.36 seconds faster**
- **45.5% lower in runtime**
- **1.83× as fast**
- Approximately **83% higher in throughput**

Memory usage remains effectively unchanged at approximately **14.46 GiB** because both versions still load the entire file into `Data`.

## Why it is faster

The naive implementation repeatedly treats `Data` as a generic Swift collection:

```swift
let slice = data[index...]

let semiIndex = slice.firstIndex(of: .semi)
let newLineIndex = slice.firstIndex(of: .newline)
```

For every reading, this involves:

- Creating `Data` slices
- Calling generic `Collection.firstIndex(of:)`
- Repeated `Data` subscripting
- Advancing collection indices
- Accessing the `Data` representation and storage
- Performing bounds and representation checks

The naive trace reflects this cost:

- `Data._Representation.subscript.getter`: **13.36%**
- `Collection.firstIndex(of:)`: **6.19% self time**, plus work beneath it
- Several `Data` storage, offset, byte, and collection-index functions among the top costs
- Approximately **32% inclusive time** under `firstIndex(of:)` in the call-tree view

The current implementation acquires contiguous storage once:

```swift
data.withUnsafeBytes { bufferPointer in
    while let reading = parseReading(
        from: bufferPointer,
        offset: &offset
    ) {
        // ...
    }
}
```

It then calculates addresses directly and uses `memchr`:

```swift
let base = pointer.baseAddress! + offset
let semiPtr = memchr(base, ...)
let newLinePtr = memchr(semiPtr + 1, ...)
```

This removes most of the generic `Data` collection machinery from the hot loop. `memchr` works directly on contiguous memory and uses Darwin's optimized implementation.

The optimization is therefore more than simply replacing `firstIndex` with `memchr`:

> Replace repeated high-level `Data` collection traversal with one contiguous buffer and direct pointer-based scanning.

## What the new trace shows

The old `Data.firstIndex`, subscripting, and storage-access functions have disappeared from the current run's major hotspots. This confirms that the intended work was removed.

The new leading costs include:

- `_platform_memmove`: **8.07%**
- `String.init(bytes:encoding:)`: **5.17%**
- String normalization and ASCII checking
- String hashing and comparison
- Floating-point parsing
- Dictionary lookup

This is a bottleneck shift: delimiter searching is now relatively cheap, exposing string construction, copying, hashing, and numeric parsing as the next major costs.

### Why `memmove` is now prominent

The current parser constructs two new `Data` values for every row:

```swift
let nameData = Data(buffer: ...)
let tempData = Data(buffer: ...)
```

Those initializers copy the city and temperature bytes. `_platform_memmove` in the trace is evidence of those copies.

For every reading, the implementation still:

1. Copies city bytes into `Data`
2. Decodes the city `Data` into a `String`
3. Copies temperature bytes into `Data`
4. Decodes the temperature `Data` into a `String`
5. Parses the temperature string as `Double`
6. Hashes and compares the city string in the dictionary

This explains both the substantial improvement and the remaining cost.

## Next optimization

The trace now supports **deferring city-name decoding** as the next step. This should remove or reduce:

- Per-row city `Data` copies
- Per-row UTF-8 decoding and validation
- String normalization
- String allocation
- String hashing and comparison

After that, parsing temperatures directly as integer tenths should remove the temperature `Data`, temperature `String`, and floating-point parser.

## Correctness issue

The current newline search length is one byte too large:

```swift
maxSearch -= cityNameCount
memchr(semiPtr + 1, ..., maxSearch)
```

Because the search begins after the semicolon, the semicolon must also be subtracted:

```swift
let bytesAfterSemicolon =
    pointer.count - offset - cityNameCount - 1
```

Otherwise, the final search can inspect one byte beyond the buffer.

## Recording the stage

The current benchmark file is named `00-naive-dirty.json` even though it contains the optimized result. Record the stage with an explicit name:

```bash
RUN_NAME=01-raw-buffer mise run benchmark
RUN_NAME=01-raw-buffer mise run profile
```
