# Amdahl's Law

Amdahl's Law is a formula used to calculate the maximum theoretical speedup of an entire program when only a portion of it is parallelized.
- **The Formula:**
  $$S(s) = \frac{1}{(1-p) + \frac{p}{s}}$$
  Where:
  - $S(s)$ is the theoretical speedup.
  - $p$ is the fraction of the program that can be parallelized.
  - $1-p$ is the serial (non-parallel) fraction.
  - $s$ is the speedup factor of the parallelized portion (e.g., number of CPU cores).
- **The Core Limit:** No matter how many CPU cores you add, the maximum speedup is strictly limited by the serial portion of the program ($1-p$). If 10% of your program is serial, the absolute maximum speedup is $10\times$, even with an infinite number of processors.

## Interview Questions & Answers

### Q1: How does Amdahl's Law affect distributed database write performance?
- **Answer:** In a distributed write pipeline, specific operations are strictly serial (e.g., consensus voting, transaction lock ordering, writing to the physical WAL file). No matter how many read replicas or storage shard nodes you add to parallelize reads, your write speedup is bottlenecked by the serial WAL execution and coordinator consensus latency, matching Amdahl's law constraints.
