# Optimization Steps

- [x] **Naive implementation**
  - Load the input with `Data`.
  - Find `;` and newline bytes with `Data.firstIndex(of:)`.
  - Decode every city and temperature as `String` values.
  - Parse temperatures as `Double` and aggregate with a standard dictionary.

- [x] **Use contiguous raw-buffer parsing**
  - Acquire the input's base pointer once rather than entering `withUnsafeBytes` for every reading.
  - Replace `Data.firstIndex(of:)` with bounded `memchr` searches.
  - Begin the newline search immediately after the semicolon.
  - Continue using `String`, `Double`, and the standard dictionary so this optimization is measured independently.

- [ ] **Defer city-name decoding**
  - Keep station names as byte-backed keys while aggregating.
  - Decode each distinct city to `String` only when producing output.

- [ ] **Parse temperatures as integer tenths**
  - Parse temperature bytes directly into an integer such as `16.2 → 162`.
  - Aggregate integer minimum, maximum, and sum values.
  - Divide by ten only while formatting the final output.

- [ ] **Memory-map the input file**
  - Replace the allocated `Data` input with a stable, read-only `mmap` region.
  - Keep the mapping alive until all parsing, merging, and output-key decoding is complete.

- [ ] **Use zero-copy station keys**
  - Represent each station with a pointer and byte count into the mapped file.
  - Compare equal-length names with `memcmp`.
  - Avoid allocating or copying station names in the hot loop.

- [ ] **Precompute station hashes**
  - Hash station bytes while parsing each row.
  - Store the hash alongside the pointer and length.
  - Compare bytes only when hash and length match.

- [ ] **Process newline-aligned chunks in parallel**
  - Divide the mapped file into independent ranges ending at newline boundaries.
  - Process ranges with a task group and task-local aggregation tables.
  - Merge task results after parsing completes.

- [ ] **Tune chunk count and parallelism**
  - Compare processor count, twice the processor count, and fixed chunk sizes.
  - Keep the best configuration based on repeated measurements rather than assumptions.

- [ ] **Use a fixed-capacity flat hash table**
  - Replace each worker's general-purpose dictionary with open addressing and linear probing.
  - Store pointer, length, precomputed hash, and aggregate values directly in each slot.
  - Convert to the final result dictionary only when merging or formatting output.
