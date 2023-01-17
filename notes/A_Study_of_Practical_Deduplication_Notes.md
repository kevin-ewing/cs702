# A Study of Practical Deduplication

## First Pass

- 4 week study of nearly one thousand Microsoft computers
- Analyzed data on disk to determine the relative efficacy of data deduplication, when comparing whole-file versus block-level elimination of redundancy
- Found that whole-file deduplication achieves about three quarters of the space saving of the most aggressive block-level deduplication for storage of live file systems, and 87% of  the savings for backup images
- Many File systems often contain redundant copies of information: identical files or sub-file regions, possible stored on a single host, on a shared storage cluster, or backed up to a secondary storage
    - Data deduplication takes advantage of this redundancy to reduce the underlying space needed to contain the file system or backup images of that same file system
- Deduplication can happen at either the sub-file or whole-file level. More fine-grained deduplication creates more opportunities for space saving, but necessarily reduces the sequential layout of some files, which may have significance performance impacts when hard disks are used for storage
- Whole-file deduplication is simpler and eliminates file-fragmentation concerns, though at the cost o some otherwise reclaimable storage.
- We find that while-file deduplication together with sparseness is a highly efficient means of lowering storage consumption, even in a backup scenario. It approaches the effectiveness of conventional deduplication at a much lower cost in performance and complexity.
    - The environment studied, despite being homogenous, shows a large diversity in file system and file sizes.

## Second Pass