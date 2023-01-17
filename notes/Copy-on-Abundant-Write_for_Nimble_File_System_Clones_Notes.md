# Copy-on-Abundant-Write for Nimble File System Clones

## First Pass

- An ideal clone implementation meets the following performance goals, together they are nimble clones:
    - Creating the clone has low latency
    - Reads are fast in all versions
    - Writes are fast in all versions
    - The overall system space is efficient
- Some copy on write heuristics can be too coarse to be space efficient or two fine-grained to preserve locality.
- Duplicating large objects is so prevalent that many file systems support logical copies of directory trees without making full physical copies. Physical copying a large file or directory is expensive - both in time and space
- A classic optimization, frequently used with volume snapshots is to implement copy-on-write (CoW).
    - Many supports block-level CoW or other implementation-specific interfaces
    - Some file systems support CoW file or directory copies via `cp --reflink`
    - Traditional CoW presents a tradeoff between write amplification and locality
    - Copy granularity is usually the largest fine-tuning
    


## Second Pass