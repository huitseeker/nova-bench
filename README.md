# Nova benchmarks

Here's a list of some of the benchmarks we've been taking to better understand how Nova performs vs other proof systems.

## Recursive hashing

Recursively hashing SHA256 k times. That is, computations of the form $h(h(h(h(h(x)))))$.

Rationale: Similar to https://github.com/celer-network/zk-benchmark but doing hashing recursively to take advantage of Nova+IVC.

Code: https://github.com/oskarth/recursive-hashing-bench

#### Proving systems

| Framework        | Arithmetization | Algorithm | Curve  | Other        |
|------------------|-----------------|-----------|--------|--------------|
| Circom (snarkjs) | R1CS            | Groth16   | Pasta  |              |
| Nova (seq)       | Relaxed R1CS    | Nova      | Pasta  |              |
| Nova (par)       | Relaxed R1CS    | Nova      | Pasta  | parallel PoC |
| Halo2            | Plonkish        | KZG       | BN254  |              |

### Prover time

#### Powerful laptop

Hardware: Macbook Pro M1 Max (2021), 64GB memory.

| k     | Circom | Nova (total) | Nova (step sum) | Halo 2 (KZG) |
|-------|--------|--------------|-----------------|--------------|
| 1     | 0.3s   | 0.2s         | 0.1s            | 0.8s         |
| 10    | 7.3s   | 2.4s         | 1.2s            | 0.8s         |
| 100   | 62s    | 24s          | 12.5s           | 1.6s         |
| 1000  | -      | 240s         | 125s            | 25s          |

#### Powerful server

Hardware: Server with 72 cores and ~350GB RAM.

| k       | Nova (step sum) | Halo 2 (KZG) |
|---------|-----------------|--------------|
| 100     | 19s             | 2.5s         |
| 1000    | 190s            | 41.6s        |
| 10000   | 1900s           | 389.1s       |
| 100000  | 19000s          | ?            |

#### Comments

This is not completely an apples-to-apples comparison, as: (i) Circom implements the recursive hashing "in-circuit", and (ii) Halo2 uses a different aritmeitization and lookup tables with a highly optimized implementation. However, it shows how a standard operation behaves when called recursively and expressed in a (somewhat) idiomatic fashion.

Step sum is the sum of all the individual folds, i.e. it doesn't account for the witness generation. The witness generation overhead is quite high, especially when running it in WASM (MBP M1 limitation). The step sum is the more representative metric. For Nova, we also don't count the SNARK verification part (currently done with Spartan using IPA-PC, not a huge overhead).

Circom (Groth16) and Nova are run with the Pasta (Pallas/Vesta) curves and use (Relaxed) R1CS arithmetization. Halo2 (KZG) is using BN254 and Plonkish arithmetization.

### Memory usage and SRS

#### Powerful server

Hardware: Server with 72 cores and ~350GB RAM

| k       |  Nova (seq) | Halo 2 (KZG) | Nova (par PoC) |
|---------|---------------|--------------|-------------|
| 100     |  1.6GB        | 3.7GB        | 9GB         | 
| 1000    |  1.6GB        | 32GB         | 244GB       | 
| 10000   |  1.6GB        | 245GB        | OOM         | 
| 100000  |  1.6GB        | ?            | ?           |

#### Comments

For Circom, at k=100 the number of constraints is 3m, and we need 23 powers of tau or structured reference string (SRS), 2^23. This is a 9GB file and it increases linearly, quickly becomes infeasible.

For Halo2, which is Plonkish, the situation is a bit better. The SRS of Halo2 is 2^18 for k=100, 2^22 for k=1000, and 2^25 for k=10000. Because the arithmetization is more efficient, Halo2 needs a shorter SRS than Circom.

Nova, assuming it is run natively or with Circom C++ witness generator, has constant memory overhead.

In the current parallel PoC of Nova there's a bug in the current parallel code that leads to linearly increasing memory. This isn't intrinsic to Nova though, and more of a software bug.

### Conclusion

Based on the above we draw three conclusions:

1. **Nova is memory-efficient.** For large circuits this matters a lot, where Circom and Halo2 both require a big SRS and run out of memory. This is especially important in low-memory environments.
2. **R1CS vs Plonkish matters a lot.** Plonkish with lookup tables leads to a much faster prover time for Halo2 (KZG) with e.g. SHA256 recursive hashing. This motivates work on alternative folding schemes, such as Sangria.
3. **Be mindful of constraints vs recursion overheard.** For SHA256 the number of constraints isn't big enough (~30k) compared to recursive overhead (~10k) to see huge performance gains. This suggests that we want to use somewhat large circuits for Nova folding. We see this next in the Bitcoin benchmarks.

## Bitcoin blocks

Prove the correct hashing of Bitcoin blocks.

Rationale: Non-trivial example, more constraints per fold (~500k-1m), existing code in both Nova and Halo (old prototype).

Code: https://github.com/oskarth/recursive-hashing-bench/ and https://github.com/ebfull/halo

Hardware: Server with 72 cores and ~350GB RAM

### Nova

*NOTE: This is for 120 blocks.*

| iteration_count | per_iteration_count | prover_time (seq)      | prover_time (par PoC) |
| ---             | ---                 | ---              | --- | 
| 120             | 1                   | 33.3s| 24.1s |
| 60              | 2                   | 28s| 18.4s |
| 40              | 3                   | 27.4s| 18.7s|
| 30              | 4                   | 24.8s| 17.6s|
| 24              | 5                   | 25.1s| 18.4s|

We notice that validating more Bitcoin blocks in an iteration speeds up overall prover time.

We see that parallel implementation is ~35% faster. Note that the current parallel PoC is quite naive and can very likely be improved by a lot.

Also note that this is still R1CS, so making it Plonkish would likely improve e.g. inner SHA256 hashing (see above).

### Halo (old prototype)

*NOTE: This is for 5 blocks.*

Context: This was prototype that, at the time, performed significantly better than previous Groth16 state of the art and convinced Zcash to invest in Halo.

Proving time ~108s

### Conclusion

**With many constraints, we see a significant 100x (77-144x) speedup over (naive)**.

## Future benchmarks

While above benchmarks gives us some insight, it'd be useful to do further benchmarks to better understand how Nova behaves. These are not in order of priority.

1) Better standard operation benchmark in Nova with more constraints that also exists in Circom/Halo2. This could be porting Halo Bitcoin to Halo2, or only doing one fold with larger preimage in SHA256, or something similar.
2) Halo2 comparable example with two column layout. This would show a more realistic comparison vs R1CS based Nova.
3) Halo2 example using recursion. Current "recursive hashing" SHA256 example uses lookup tables only.
4) Better parallel comparison. Currently parallel implementation only shows 35% improvement + memory increase. We expect this can be done a lot better. (Parallelization can partially be simulated with one thread vs many threads on a single machine).
6) GPU comparison. GPU should show significant improvement vs CPU, but so far we've not been able to get it to work / show big improvement.
7) FPGA comparison. Assuming only MSM + addition operations this could lead to massive improvements. Perhaps limited to only benchmarking MSM + additions to see if this investment makes sense.
8) Different curves. Currently we use pasta curves for Nova vs BN254. It might be useful to compare different curves here, as Nova doesn't require FFT-friendly curves.
