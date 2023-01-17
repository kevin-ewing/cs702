# Characterizing the Efficiency of Data Deduplication for Big Data Storage Management

## First Pass

 - Demand for data storage and processing is increasing at a rapid speed in the big data ara. Such high amounts of data pushes the limits on storage capacity and on storage networks. A significant portion of the dataset in big data workloads is redundant. As a result, deduplication technology, which removes replicas, becomes an attractive solution to save disk space and traffic on a big data environnement
    - However the overhead of extra CPU computation (hash indexing) and IO latency introduced by deduplication should be considered.
    - Therefore the net effects of deduplication need ot be examined
- Advantages are:
    - saving  disk space, which further leads to saving money on buying storage devices and
    - reducing IO traffic, which results in higher IO throughput. 
- Cloud vendors pay more attention to deduplication because there are many replicas in the data outsourced to the cloud (75% of data are redundant)
- Conclusion:
    - Local deduplication can leverage parallelism to reduce the overhead of hashing but cannot exploit the maximum redundancy
    - Global deduplication can eliminate all redundant disk accesses byt cannot hide the hash overhead
    - Fine-grained deduplication is not always better, especially when the dataset is big
    - Although deduplication brings extra power consumption, the energy consumption can be less if the level of redundancy is high enough to reduce execution time

## Second Pass