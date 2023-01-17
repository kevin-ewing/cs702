# Alternatives for Detecting Redundancy in Storage Systems Data

## First Pass

- Identifying duplicate copies of data is a crucial issue for data deduplication. They talk about three methods used to discover identical portions of data: 
    - Whole file content hashing
    - Fixed size blocking
    - Chunking strategy that uses Rabin fingerprints to delimit content-defined data chunks
- Computer systems don't just store several copies of the same data, but they often manipulate these copies regularly
    - Some applications may generate versions of a document stored as separate files, byt whose content differs only slightly
- Over the past few years there has been a constant reduction in the cost of raw disk storage
    - Some may then argue that the space saving obtained by suppressing identical portions of data are of minimal significance, however there are other factors to be considered
        - Storage systems exploit deduplication patterns to optimize the use of storage space and band width
            - Single Instance Storage (SIS)
        - File systems may obtain improved caching performance if they are aware of contents shared between files. May be possible to provide better hit ratios for a given cache size
        - In mobile environments, devices are often limited in storage and bandwidth as well as the further factors such as energy consumption and network costs
- "As may be expected, the whole-file approach was always at the bottom of the ranking."
- The whole file content approach reported modest levels of similarity with exceptionally high values in research groups’ data (25%) and expanded source code distributions (96.86%). In data sets with a not-so-evident correlation such as the sunsite.org.uk mirror the similarity levels plummeted (5%).
- Conclude there is no one method able to perform satisfactorily on all data sets.


## Second Pass