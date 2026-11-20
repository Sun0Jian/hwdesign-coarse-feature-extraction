# Streaming Coarse Feature Extraction

**GitHub repository:** https://github.com/Sun0Jian/hwdesign-coarse-feature-extraction

## 1. Project Team

Jian Sun

## 2. IP Definition

**Purpose and research connection.** We will build an HLS accelerator that converts ordered one-dimensional data into compact block features. This implements the spatial averaging and coarse-state construction step in a reduced-model workflow. It supports the research question of what information a simplified representation retains when detailed data are averaged. The course project uses synthetic integer sequences and requires only addition, subtraction, comparison, and shifting.

**Input and output.** The input is an array of N unsigned 16-bit samples, with 16 <= N <= 65536 and N divisible by 16. Each consecutive group of 16 samples is one independent block. Sample order defines adjacency. For each block, the IP produces three unsigned features:

```text
mean      = sum(x[0:16]) >> 4
range     = max(x[0:16]) - min(x[0:16])
variation = sum(abs(x[j] - x[j-1]) for j = 1,...,15)
```

The mean is rounded down; its error relative to the exact mean is less than one input unit. Differences never cross block boundaries. Compare-then-subtract implements absolute differences without unsigned underflow. The sum and variation use 20-bit accumulators, sufficient for their maxima of 1,048,560 and 983,025. Each output is one 64-bit word: mean in bits [15:0], range in [31:16], and zero-extended variation in [63:32]. A 32-byte input block therefore becomes an 8-byte feature record.

**Illustrative experiment.** Eight zeros followed by eight 16s and an alternating sequence of 0 and 16 both have mean 8 and range 16. Their variations are 16 and 240. This demonstrates selected information preserved by an additional feature. These features are candidate summaries; they do not uniquely describe a sequence or establish physical prediction accuracy.

**Host interaction.** The PS places the input in shared memory and allocates a separate output buffer of N/16 words. AXI4 memory-mapped ports read the input and write the features. AXI4-Lite registers carry the buffer addresses, a 32-bit N, start/done control, and a status value. Invalid N is rejected before memory access. The host waits for completion before reusing buffers or changing configuration.

**Hardware motivation and scope.** The computation has sequential memory access, constant working state, and simple repeated operations that can be pipelined while reading and writing overlap. We will measure throughput and resource cost; the smaller output size is not a promised runtime speedup. The required result is a verified HLS design with synthesis and RTL co-simulation. Initial synthesis targets xc7z020clg400-1 at 100 MHz. Board deployment is optional.

## 3. IP Architecture

The top function validates the request, then runs four tasks concurrently in one HLS DATAFLOW region. Internal channels are finite hls::stream FIFOs.

| Module | Responsibility | Interface and payload |
|---|---|---|
| Memory reader | Read exactly N samples in order. | AXI4 memory-mapped input to a 16-bit sample FIFO. |
| Difference stage | Pair each sample with its absolute difference from the preceding sample; use zero for the first sample of each block. | 16-bit input FIFO to a 32-bit FIFO carrying sample and difference. |
| Block reducer | Accumulate sum and variation, track minimum and maximum, and emit one packed record after every 16 samples. | 32-bit input FIFO to a 64-bit feature FIFO; exactly N/16 records. |
| Memory writer | Store all feature records in block order. | 64-bit feature FIFO to AXI4 memory-mapped output. |

All counters and state reset for each invocation. Block statistics reset every 16 samples. Blocking stream operations preserve order under backpressure, and completion follows all required output writes. The design initially targets an initiation interval of one sample per cycle; achieved II and FIFO behavior will be checked rather than assumed.

**Development and verification.** We will first implement an integer Python reference and a straightforward single-loop HLS baseline. We will then build the four-stage design and compare both on identical inputs. Module tests and C/RTL comparisons will check exact feature values and output counts. Cases include zero, constant, maximum-value, monotonic, alternating, and seeded random data; adjacent blocks with different endpoints; invalid lengths; and repeated invocations. RTL checks will cover completion with finite FIFOs and stalls. Matched-mean examples and a pair with identical feature records will illustrate what the representation retains and loses.

**Evaluation and deliverables.** Report cycles, achieved II, throughput across several batch sizes, and LUT/FF/BRAM/DSP use for the baseline and pipelined versions at the same target clock. Measure startup overhead and memory stalls separately where available. Deliver the reference, HLS sources, testbench, build scripts, and readable result tables. Separate modules support incremental testing of memory access, neighbor differences, block arithmetic, and packing. The planned sequence is reference and baseline, pipeline integration, then RTL verification and evaluation over three to four weeks.

Course reference: [Initial project planning requirements](https://sdrangan.github.io/hwdesign/docs/projects/planning.html).
