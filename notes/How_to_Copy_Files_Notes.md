# How to Copy Files (SIMILAR TO COPY-ON-ABUNDANT-WRITE)

## First Pass

- Making logical copies, or clones, of files and directories is curial to many real-world applications and workflows
    - Backups, virtual machines, and containers
- Goals include (If all 4 are upheld then it is said to be a nimble clone)
    - Low latency
    - Reads are fast in all versions (spacial locality is always maintained even after modifications)
    - Writes are fast in all versions
    - Overall system is space efficient

## Second Pass