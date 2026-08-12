# Optimization State and Research Log

Last updated: 2026-08-12

This file is the source of truth for the current kernel, validated results,
failed experiments, mathematical constraints, and the next research directions.
Results from disposable `/tmp` prototypes are explicitly separated from the
stable workspace implementation.

## 1. Goal and hard constraints

Target workload:

```text
(forest_height, n_nodes, batch_size, rounds) = (10, 2047, 256, 16)
VLEN = 8
N_CORES = 1
SCRATCH_SIZE = 1536 words
```

Machine capacities per cycle:

```text
ALU=12, VALU=6, load=2, store=2, flow=1
```

Hard rules:

- Never modify `tests/`, `problem.py`, or the frozen simulator to improve a
	score.
- Validate against `tests/submission_tests.py`; the local simulator is useful
	for iteration but is not the final authority.
- A scheduler result is invalid if it stops early, reports `STUCK`, or schedules
	fewer operations than were emitted, even if the printed bundle count is low.
- Use container system Python (`/bin/python3`); no virtual environment is
	required.
- Offline tools such as OR-Tools and Z3 must never become runtime dependencies
	of the submitted kernel.

## 2. Current workspace checkpoint

The current workspace source of truth is `perf_takehome.py`:

| Metric | Value |
| --- | ---: |
| Frozen-shape cycles | **982** |
| Emitted/scheduled IR operations | **12045 / 12045** |
| Baseline | 147734 |
| Speedup | **150.44x** |
| Scratch | **1523 / 1536 words** |
| Spare scratch | 13 words |

The specialized kernel is selected only for `(10, 2047, 256, 16)`. Other
shapes use the scalar fallback.

The official suite currently passes:

```text
/bin/python3 tests/submission_tests.py
Ran 9 tests in 3.376s
OK
CYCLES: 982 on all nine executions
```

The prior VS Code Copilot session store was searched after moving machines. It
preserved the full path through the integrated 999-cycle checkpoint and a later
status report for 991, but not the disposable `/tmp` artifacts from the old
machine. The 991 workspace state was independently reproduced before starting
new experiments: 12048/12048 operations, 1524 scratch words, and all nine
official tests passing.

Additional fresh-process validation on 2026-08-12:

```text
seeds: 1, 2, 3, 7, 42, 123, 999, 20260811, 20260812
cycles: 982 for every seed
correct: true for every seed
ops: 12045 complete
peak scratch: 1523 / 1536

git diff --exit-code origin/main -- tests/ problem.py
git diff --check
(both commands produced no output)
```

The commands are recorded in section 11 and must be rerun whenever the stable
kernel changes.

### 2.1 Previous disposable compiler checkpoint

The previous best experimental schedule was isolated under `/tmp/vliw-1063`:

| Metric | Value |
| --- | ---: |
| Scheduled cycles | **1004** |
| Emitted IR operations | **12184** |
| Peak scratch | **1521 / 1536 words** |
| Spare scratch | 15 words |
| load slots | 1907 |
| VALU slots | 5811 |
| ALU slots | 11720 |
| flow slots | 836 |
| store slots | 44 |

Resource floors for this candidate are:

```text
load  ceil(1907 / 2)  = 954
VALU  ceil(5811 / 6)  = 969
ALU   ceil(11720 / 12) = 977
flow  836
store ceil(44 / 2)    = 22
maximum floor          = 977 cycles
```

Validation level as of 2026-08-11:

- Every emitted operation was scheduled; there was no `STUCK` or truncated
	execution.
- The schedule ran correctly in `tests/frozen_problem.py` for the frozen shape
	and seed 123.
- The result was reproduced in a fresh Python process at 1004 cycles.
- This 1004 result was superseded first by the integrated 999-cycle kernel and
	then by the current 983-cycle workspace kernel.

The exact experimental architecture is:

- Named SSA-like IR with online dual-pool scratch allocation.
- Last-use destructive coalescing with explicit ownership transfer.
- 20 concurrent groups and a 20+12 wave structure.
- Second-wave slots are released after round 10.
- Release pairing:

```text
(11, 9, 5, 3, 10, 8, 7, 6, 4, 1, 2, 0)
```

- Global C6-XOR state encoding.
- Tree depths 5 and 6 are C6-encoded once in memory using lane ALU XORs;
	deep scatter rounds then omit their per-group `scat_encode` operation.
- Round 2 and round 3 `hash1_a` branches are statically lowered to lane ALU;
	`(round 5, group 10)` is also lowered.
- Final round depth-4 lookup mask:

```text
{0, 1, 3, 11, 14, 18, 21, 22, 24, 25, 26}
```

- Final-round depth-4 bit order is reversed only for groups 3 and 26.
- Dynamic VALU-to-ALU split offload remains enabled.

The exact priority parameters for the 1004 result are retained because
small changes can move the result by tens of cycles:

```text
group priority slope:       -0.2 * group
extra group >= 25:          -7.5
extra group >= 26:          -4.0
group 29 / 30 / 31:         -1.0 / +0.5 / +1.0
round 14 / round 15:        -0.5 / -1.0

engine costs:
  load=0.25, flow=0.30, ALU=0.05, VALU=0.005
late flow cost:             3.0
emit-order scale:           15.0
proactive offload cutoff:   125.0

round-15 group-engine slopes for groups 24-31:
  load=-0.25 * (group - 24)
  ALU =-0.50 * (group - 24)
  VALU=-0.10 * (group - 24)
```

The controlling scratch prototype was `/tmp/vliw-1063/perf_takehome.py`.
`/tmp/vliw-1063/best_1006_config.py` captures the preceding 1006 parameter
set; the 1004 change was `ENGINE_COSTS["alu"]: 0.125 -> 0.05`.

### 2.2 Decisive 999-cycle change

The final improvement shares the depth-4 lookup inputs with tree-memory
preencoding:

1. Load the two depth-4 eight-node blocks once for the cached lookup table.
2. XOR those vectors with C6 in place.
3. Build the mirrored depth-4 lookup table directly from the encoded vectors,
	avoiding a second per-source C6 transform.
4. Store those same vectors back to tree memory.
5. Depth-4 scatter groups then consume encoded memory and omit their per-group
	`scat_encode` operations.

The winning policy is:

```text
PREENCODE_TREE_DEPTHS = {4, 5}
PREENCODE_TREE_WITH_ALU = False
SHARE_DEPTH4_LOOKUP_PREENCODE = True
```

The integrated runtime is self-contained in `perf_takehome.py`. It does not
import OR-Tools, Z3, MCP components, `/tmp` modules, or search helpers. MCP was
not needed for implementation or validation; one subagent's MCP wait was an
incorrect tool-routing choice and was bypassed in favor of local execution.

### 2.3 Decisive 983-cycle change

The 2026-08-12 search started from the frozen-correct 991 checkpoint. Its tail
was load-saturated through cycle 980, after which group 29's final hash and
store completed at cycle 990. Scheduler-only priority changes merely moved this
tail between groups 29 and 31.

The winning change modifies the earlier round-4 depth-4 lookup shape instead:

```text
round-4 vselect groups added: 16, 26, 29
round-4 reversed-bit groups:  16, 26, 29
existing round-15 reversal:   group 3
physical group-order swap:    positions holding groups 5 and 25
```

Exact retained configuration:

```text
round-4 vselect mask:
{0, 1, 3, 11, 12, 13, 15, 16, 18, 21, 22, 24, 25, 26, 27, 28, 29, 30}

round-15 vselect mask:
{0, 1, 3, 11, 14, 18, 19, 21, 22, 24, 25, 26}

round/depth bit reversals:
{(4, 16), (4, 26), (4, 29), (15, 3)}

logical-to-physical group order:
(0, 1, 2, 3, 4, 25, 6, 10,
 8, 9, 7, 11, 12, 13, 26, 15,
 16, 17, 18, 19, 20, 21, 22, 23,
 24, 5, 14, 27, 28, 29, 30, 31)
```

Measured progression from the same 991 baseline:

```text
988  add round-4 lookup group 16 alone
988  add round-4 lookup group 26 alone
986  add round-4 lookup groups 16, 26, and 29
985  also reverse those three groups' bit order
983  additionally swap physical groups 5 and 25 in emission order
```

The three extra lookups remove 24 scalar loads. Reversing their lookup bit
order lets each lookup start from its earliest retained branch bit, while the
group-order swap changes which physical work occupies the two-wave schedule's
critical contexts. The final load now occurs at cycle 971, final VALU at 979,
and final store at 982.

### 2.4 Final-store drain improvement to 982

At 983 cycles, the final six stores completed as `1+2+2+1` across cycles
979-982 because groups 26 and 27 did not both finish their final hash combine
by cycle 978. A narrow operation-ID priority bonus fixes that phase:

```text
g26_r15_hash5_combine: +15.0
g27_r15_hash5_combine: +15.0
```

Both combines now complete at cycle 978. The six dependent stores issue as
`2+2+2` in cycles 979-981, reducing makespan to 982 without changing emitted
operations, slot totals, or peak scratch. Targeting either operation alone, or
using broad group/round/engine bonuses, did not improve the schedule.

## 3. Active architecture

### 3.1 SIMD and tiling

- The 256 values are represented as 32 vectors of 8 lanes.
- Values are loaded once, kept resident in scratch for all rounds, and stored
	once at the end.
- There are 20 private SIMD contexts. The 32 vectors are processed as waves of
	20 and 12 contexts.
- Each context owns `path`, `tmp1`, `tmp2`, and three retained path-bit vectors.
- The 20-context point is the best measured balance between independent work
	and the 1536-word scratch limit.

### 3.2 Hash dataflow

The six reference stages are reduced to the following main vector dataflow:

1. Stage 0: one `multiply_add` with multiplier 4097.
2. Stage 1: constant XOR, right shift by 19, and combine XOR.
3. Stages 2+3: two independent `multiply_add` operations followed by one XOR.
4. Stage 4: one `multiply_add` with multiplier 9.
5. Stage 5: right shift by 16 and combine XOR.

The stage-5 constant `C6 = 0xB55A4F09` is omitted from the main hash chain.
The live value is therefore C6-XOR-encoded across rounds. This provides two
benefits:

- Stage 5 costs two operations instead of three.
- The encoded parity is the complement of the reference parity, so the stored
	path is a mirrored path and can be used consistently for cached lookup.

Raw deep-tree nodes are not encoded, so a deep round decodes the value once
before XOR with the gathered node. Final output is decoded before store.

### 3.3 Tree access

- Depths 0-3 are cached as broadcast vectors and selected with retained branch
	bits.
- Depths 4-10 normally use scalar gather: eight ALU address operations, eight
	scalar loads, and scalar XORs that can overlap with vector hash work.
- On round 4, 18 vectors use the 16-entry table and 14 use direct gather. On
	round 15, 12 vectors use the table and 20 use direct gather. The asymmetric
	assignment balances saved loads against VALU/flow pressure.
- The final table uses base/difference pairs. `multiply_add` evaluates the pair
	leaves, then a small `vselect` tree chooses the result.

### 3.4 Static scheduling

The scheduler builds physical scratch dependencies:

- RAW: consumers depend on the last writer.
- WAW: writes to the same physical word are ordered.
- WAR: a new writer waits for readers of the previous value.

It then uses a critical-path and descendant-pressure priority:

```text
critical_path
+ 64 * load_pressure
+ 8  * valu_pressure
+ 4  * alu_pressure
+ 12 * flow_pressure
```

Current physical graph measurements:

```text
operations:    20275
edges:         85186
critical path: 426
```

## 4. Current roofline and the 900-cycle gap

Exact current slot totals:

| Engine | Slots | Capacity/cycle | Hard slot floor | Cycles with use |
| --- | ---: | ---: | ---: | ---: |
| load | 1910 | 2 | **955** | 959 |
| VALU | 5784 | 6 | **964** | 1160 |
| ALU | 11663 | 12 | **972** | 1068 |
| flow | 888 | 1 | **888** | 888 |
| store | 32 | 2 | 16 | 32 |

The current algorithm cannot reach 900 by scheduling alone. At 900 cycles the
individual budgets are:

```text
load <= 1800       current excess: 110 slots
VALU <= 5400       current excess: 384 slots
ALU  <= 10800      current excess: 863 slots
flow <= 900        current headroom: 12 slots
```

An optimistic combined ALU/VALU bound treats one ordinary vector operation as
eight scalar lane operations:

```text
scalar-equivalent work = 11663 + 8 * 5784 = 57935
combined capacity       = 12 + 8 * 6 = 60 lane-ops/cycle
optimistic floor        = ceil(57935 / 60) = 966 cycles
```

This bound is optimistic because vector `multiply_add` does not map to one
scalar ALU operation. It nevertheless gives a useful structural target.

There are exactly 32 vector groups * 16 rounds = 512 hash invocations. Removing
one true vector operation from every hash saves 512 vector slots, or 4096
scalar-equivalent lane operations. That is almost exactly the computation gap
between the current architecture and 900 cycles.

The load target requires at least 110 additional load-slot savings, equivalent
to removing 14 full eight-lane gather groups (112 loads). Because flow has only
12 cycles of headroom, those gathers cannot simply be replaced by long
`vselect` chains.

Practical conclusion: a legitimate 900-cycle design most likely needs both:

1. A nine-operation equivalent hash or an equally large cross-round fusion.
2. A factorized lookup that removes at least 14 gather groups without adding a
	 comparable VALU/ALU/flow cost.

### 4.1 Current 982 roofline

The integrated scratch-aware SSA compiler has these exact slot totals:

```text
load   1863 slots, floor 932
VALU   5843 slots, floor 974
ALU   11585 slots, floor 966
flow    857 slots, floor 857
store    38 slots, floor 19
maximum hard floor: 974 cycles
```

The remaining 8-cycle gap above the aggregate floor is now primarily the final
round's VALU dependency chains. The dependent stores are fully packed at two
per cycle once their producers finish. The new lookup policy deliberately
trades flow headroom for fewer loads; VALU is the hard resource floor.
Reaching 900 still requires structural work rather than scheduler weights
alone.

## 5. Optimization progression

Stable checkpoints reached during this effort:

```text
147734  scalar baseline
	1333
	1312
	1289
	1285
	1275
	1228
	1226
	1216
	1198
	1178  former physical-scheduler checkpoint
```

Disposable scratch-aware compiler checkpoints:

```text
1063  reproduced public SSA/compiler baseline; nine tests passed
1082  global-C6 branch; nine tests passed
1085  correct last-use/coalescing branch; nine tests passed
1035  complete aggressive schedule with round-based group release
1031  one-time depth-5 tree preencoding
1030  ALU tree preencoding and scheduler rebalance
1029  nontrivial second-wave release pairing
1025  static hash1_a lowering for rounds 2 and 3
1023  add static hash1_a lowering at (round 5, group 10)
1021  group-level hash lowering placement
1018  linear group-priority compensation
1016  targeted group/round priority shaping
1014  improved second-wave release pairing
1011  retuned final depth-4 lookup assignment
1010  second lookup-mask refinement
1009  per-group final lookup bit-order reversal
1008  second release-pair refinement
1006  joint random scheduler-parameter search
1004  lower ALU critical-path weight from 0.125 to 0.05
	 999  share depth-4 lookup vectors with C6 memory preencoding
	 991  recovered workspace checkpoint; nine tests passed
	 986  add round-4 lookup groups 16, 26, and 29
	 985  reverse the three new round-4 lookup paths
	 983  swap physical groups 5 and 25; nine tests passed
	 982  prioritize two final hash combines; stores drain 2+2+2
```

The 982 result is integrated in the workspace and passes the full official
suite.

The main retained improvements are:

- SIMD value residency and one-time input/output traffic.
- Static fixed-shape unrolling.
- Stage-0/stage-4 `multiply_add` fusion.
- Algebraic stage-2/stage-3 fusion.
- C6-XOR-encoded state.
- Retained path bits and mirrored path representation.
- Cached depth-0 through depth-3 tree nodes.
- Scalar deep gather overlapped with vector hash work.
- 20-context tiling.
- Asymmetric round-4 and round-15 depth-4 lookup assignment.
- Resource-aware dependency scheduling.

## 6. Measured experiments and dead ends

All cycle numbers below are correct executions unless explicitly marked
invalid or incorrect. Regressions were reverted from the workspace.

### 6.1 Scheduler-only experiments

| Experiment | Result | Conclusion |
| --- | ---: | --- |
| Broad priority-weight sweeps | no promoted win | Current priority is locally robust. |
| Opportunistic ready VALU-to-ALU offload | 1227 | Greedy post-selection offload damages critical ordering. |
| CP-SAT on physical DAG, horizon 1000 | infeasible | Physical anti-dependencies make 1000 impossible. |
| CP-SAT on physical DAG, horizon 1050 | infeasible | Same conclusion at 1050. |
| CP-SAT at 1100/1125, 180 s | unknown | Search did not produce a better valid schedule. |
| Tail-only CP-SAT windows | no win | Optimizing only the tail did not move the global makespan. |

Scheduler tuning cannot cross the current 972-cycle resource floor.

### 6.2 Scratch, SSA, and register allocation

| Experiment | Result | Conclusion |
| --- | ---: | --- |
| Full-vector SSA prototype | peak >=2144 words | Does not fit 1536-word scratch. |
| Simple SSA list schedule | about 1493 cycles | Renaming alone is not sufficient. |
| Physical dependency graph | 85186 edges | WAR/WAW dependencies are a major scheduling constraint. |
| Online SSA allocator prototypes | invalid/incomplete | Ownership and liveness must be proven, not inferred from a low printed cycle count. |

Important lesson: removing a dead write can extend the lifetime of an earlier
version and make WAR pressure worse. Fewer operations do not automatically mean
fewer cycles under physical scratch reuse.

A correct future SSA implementation should use this order:

1. Build a lane-precise virtual-register IR.
2. Schedule the full RAW graph with explicit group-concurrency gates.
3. Compute live intervals from the final schedule.
4. Perform offline linear-scan or interval allocation.
5. Reject the schedule if peak scratch exceeds 1536.
6. Rewrite virtual addresses only after allocation succeeds.

### 6.3 Tree lookup and address experiments

| Experiment | Result | Conclusion |
| --- | ---: | --- |
| Separate final-table workspace | 1343 | Longer table-register lifetimes destroy overlap. |
| Prepare final table much earlier | regression | Setup timing is secondary to register lifetime. |
| Reverse final lookup bit order | regression/extra deps | Earlier leaves create new table/write dependencies. |
| Full static depth-3/depth-4 lookup | 1310 | Load drops to 1584, but ALU/VALU floors exceed 1000. |
| Absolute deep address recurrence | 1296 | Correct, but serializes address state across rounds. |
| Fused deep parity/path update | incorrect | High value bits polluted the retained path invariant. |
| Broader depth-5 static lookup | regression | Selection work costs more than the saved loads. |
| Remove final table dead writes | regression | Extends previous-version WAR lifetime. |

The current final lookup is a deliberately asymmetric compromise: 26 table
lookups and 6 gathers. Aggregate slot reduction alone is not enough; the shape
of the final dependency tail matters.

### 6.4 Hash algebra and SMT results

Retained identities:

- Stages 2+3 can be represented by two MADD branches and one XOR.
- Stage 5 can omit C6 when the cross-round state is consistently XOR-encoded.

Rejected or unresolved identities:

| Search | Result |
| --- | --- |
| Absorb stage-1 C2 XOR into the two fused MADD constants | unsat on representative/full constraints |
| Represent the late hash with two affine XOR terms | unsat |
| Represent the final late hash with three affine XOR terms | unsat |
| Close C6 encoding through stage-4 MADD with alternate constants | unsat |
| Simple fixed XOR masks propagated through stage 0 | no useful solution |
| Four simple masks propagated through the full stage-5 loop | unsat at stage 0 |
| General nine-operation whole-hash synthesis in Z3 | solver returned unknown/timeouts |

Counterexample-guided synthesis found a valid complement representation for
the early hash:

```text
input mask = 0xffffffff
adjusted stage-0 constant = 0x812ab2ea
output mask = C2
```

This identity is exact, but it still requires the C2 output XOR and therefore
does not remove an operation.

Small-width enumeration showed that XOR masks which pass freely through odd
MADD stages are extremely restricted: essentially zero, the sign bit, all
ones, and the low-bits mask. The masks required to eliminate stage-1 C2 are
complex and do not pass through stage 0 in one MADD. This explains why simple
fixed-mask encodings repeatedly fail.

### 6.5 Invalid low-cycle results to ignore

One disposable dynamic-IR search printed 900 cycles, but the scheduler also
reported:

```text
SCHEDULER STUCK
done 7819 / 11740 operations
```

This is an invalid truncated program, not a 900-cycle kernel. Future search
automation must check operation completion and correctness before ranking a
candidate.

### 6.6 Scratch-aware SSA/compiler experiments on 2026-08-11

Retained mechanisms:

| Mechanism | Effect / conclusion |
| --- | --- |
| Delete stale reusable-vaddr mappings | Fixed use-after-free mappings when a virtual address was reused after its physical block had been released. |
| Separate mapping from ownership | A mapped address is not necessarily owned by the old writer; ownership must move to the newest in-place/coalesced definition. |
| Last-use destructive coalescing | Keeps the full schedule within 1536 words and removes avoidable output allocations. |
| Mid-kernel group release | Releasing the second wave after round 10 is much better than waiting for final stores. |
| Nontrivial release pairing | Pairing second-wave groups with selected first-wave release slots improved 1030 to 1029 and later 1009 to 1008. |
| One-time depth-5/depth-6 C6 encoding | Trades spare startup load/store/ALU capacity for fewer per-group deep scatter corrections. |
| Selective static `hash1_a` ALU lowering | Whole-round lowering for rounds 2-3 plus one group-level point improved scheduler balance. |
| Selective final lookup assignment | The identity of lookup groups matters even at identical aggregate slot counts; mask search improved 1014 to 1010. |
| Per-group lookup bit reversal | Global reversal regressed, but reversing only groups 3 and 26 improved 1010 to 1009. |
| Group/round/engine-specific priorities | Combined with offload and allocation order, these moved the same IR from 1021 to 1006. |
| Shared depth-4 lookup/preencoding vectors | Reused two lookup loads and encoded vectors for memory preencoding; produced the final validated 999-cycle kernel. |
| Earlier round-4 lookup expansion | Adding groups 16, 26, and 29 removed 24 late-contending loads and improved 991 to 986. |
| Per-group early-bit lookup start | Reversing only the three new round-4 lookup paths improved the new schedule to 985. |
| Context-to-physical-group reassignment | Swapping physical groups 5 and 25 improved 985 to the validated 983 checkpoint. |
| Exact final-producer priority | Prioritizing only groups 26 and 27 final `hash5_combine` operations packs the six trailing stores into three cycles and improves 983 to 982. |

Important measured dead ends or neutral results:

| Experiment | Result / conclusion |
| --- | --- |
| Preencode depths 7 or 8 | Startup load/store/ALU cost and memory barriers exceeded the saved scatter corrections. |
| Preencode depth 4 | Correct with explicit cache-read-before-store dependencies, but no makespan improvement. |
| Release after tree access instead of round tail | Correct, but worsened resource contention. |
| 17-25 concurrent contexts | 20 remained best after retuning. |
| Round-major emission windows | All tested windows regressed relative to group-major emission. |
| Global nonzero VALU critical-path weight | Regressed; the useful changes were targeted group/engine bonuses. |
| Broad mapped/coalesced retro-offload | Safe and reduced VALU slots, but did not reduce makespan without selective policy. |
| Typed scratch-address conflict extraction | Correctness improvement for the offload guard, but no direct cycle change. |
| Full hash stage moved to lane ALU | Correct but longer due to allocation and lane-completion dependencies. |
| Vector scatter only for final groups | Did not beat scalar lane XOR on the best scheduler phase. |
| Add group 29 as a twelfth final lookup | Regressed the 1004 phase to 1010 despite shortening group 29's gather path. |
| Explicit physical group permutation on the old IR | Single swaps did not beat the old schedule, but became useful after the 2026-08-12 lookup-shape change. |
| Runtime priority recomputation sweep | After fixing a default-argument binding bug, intervals 0-1000 still did not beat 1004. |
| Conservative physical rescheduler | Correct but produced 1052 cycles, worse than the online SSA scheduler. |
| Physical-DAG CP-SAT at 999 | Presolve proved this particular 1014 physical allocation infeasible; this is not a proof against other SSA allocations. |
| Physical-DAG CP-SAT at 1013 | Unknown after 120 seconds. |

The current evidence is that aggregate resource work is already sufficient for
sub-1000. The remaining blocker is the interaction among allocation order,
dynamic offload, final lookup assignment, and the final-round hash tail.

### 6.7 Parallel 991-to-983 search on 2026-08-12

Five GPT-5.6 Sol subagents independently studied the scheduler tail,
lookup/release masks, allocator/offload policy, structural operation savings,
and checkpoint provenance. All experiments were read-only or disposable until
the winning constants were promoted.

The depth-4 search evaluated 3199 focused configurations:

```text
complete schedules:          2977
rejected STUCK schedules:     222
complete candidates <= 986:   336
frozen validation:              4 seeds for every <=986 candidate
best promoted result:          983
```

A narrow follow-up from the 983 checkpoint found two independent 982-cycle
mechanisms: swapping group-order positions 0 and 8, or prioritizing the two
final producers above. They did not compose to 981. The producer-priority
variant was retained because it keeps peak scratch at 1523 instead of 1524 and
directly addresses the final store drain.

Important controls and negative results:

- Every candidate was ranked only after emitted and scheduled operation counts
	matched and scratch stayed at or below 1536.
- Truncated schedules as low as 782-956 cycles completed only 9414-10569
	operations and were rejected.
- Broad cutoff, recomputation, engine-cost, allocator, offload-family, and
	targeted-priority sweeps did not improve the original 991 IR.
- Prioritizing group 29 moved it earlier but transferred the final tail to
	group 31, leaving 991 unchanged.
- Replacing seven early final-round MADD lookup leaves with equivalent
	`vselect` leaves reduced the hard floor by one cycle but left makespan at 991.
- The winning result therefore came from changing earlier load demand and
	context assignment, not another scheduler-weight search.

## 7. Public comparison and independently reproduced results

Public data was used for architectural comparison only. No external solution
file was copied into the workspace implementation.

### 7.1 Leaderboard observations

At the time of this log, the Paradigm leaderboard API showed:

```text
908  @zartbotF
921  @adrianleb
923  @josusanmartin
950  @dougallj
957  @junlin_ai
958  @SaifAlHarthi
```

The previously mentioned 888-cycle result was not independently verified in
the public leaderboard data inspected here. The best directly observed public
server-validated score was 908.

The 908 author publicly reported an earlier 987-cycle checkpoint but did not
publish source. Their GitHub fork contains only the upstream baseline.

Public discussion independently confirmed that at least one 962-cycle result
was legitimate and did not rely on an exploit. This confirms that sub-1000 is
possible under the real rules.

### 7.2 Public 1063 implementation

A public 1063-cycle implementation was cloned only into `/tmp` and reproduced
against its frozen official suite:

```text
cycles:       1063
emitted ops:  11892
peak scratch: 1465 / 1536
tests:        9 passed
```

Its final slot totals were:

```text
load=1991, VALU=5999, ALU=11964, flow=798, store=32
```

The important architectural difference is not a lower raw slot count. It uses:

- A named SSA-like IR.
- Greedy list scheduling with dynamic scratch allocation.
- Selective VALU-to-ALU decomposition.
- In-place ownership transfer for some virtual registers.
- Explicit group and round gates to keep the live set within scratch.

This validates the general compiler direction, but its resource floors are
near 1000 and it is not itself a 900-cycle architecture.

A disposable global-C6 variant of that IR passed the official eight-run
correctness test at 1082 cycles. Scheduler retuning produced a complete
1072-bundle schedule, but that variant was not promoted or fully revalidated.
It lowered the resource floor but did not provide a path to 900.

### 7.3 Public technique summary

The strongest public high-level guidance found was:

- Use an SSA IR.
- Apply algebraic simplification.
- Use semi-greedy VLIW bundle packing.
- Use greedy register allocation.

This matches the local evidence: the current physical scratch graph is too
constrained for pure scheduler tuning, while unconstrained SSA exceeds scratch.
The missing implementation is a correct scratch-aware SSA scheduler/allocator,
not another priority constant sweep.

## 8. Prioritized next directions

### Immediate P0: continue from 982 toward 900

1. Preserve the validated 982 checkpoint before any further search.
2. Revisit shared preencoding for depth 6 without duplicating loads or vectors;
	the successful depth-4 change shows that sharing, not preencoding alone, is
	the useful abstraction.
3. Target the 8-cycle gap above the 974 resource floor with final-round VALU
	chain changes rather than broad priority sweeps; store drain is now packed.
4. Continue nine-operation hash synthesis as the main route toward 900.
5. Rerun the complete promotion checklist after every workspace change.

### Completed P0: promote the compiler independently

The scratch-aware compiler is integrated into the workspace. Completed checks:

1. Typed IR and allocator invariants are present in `perf_takehome.py`.
2. Solver/search scripts remain outside runtime code.
3. Winning parameters are ordinary module constants.
4. Official frozen tests and `problem.py` are unchanged.
5. All nine official tests and fresh-process seed checks pass.

### P0: Synthesize a true nine-operation hash

Expected value: very high. One operation removed from all 512 vector hashes is
approximately the entire compute reduction required for 900.

Next implementation steps:

1. Install or use a dedicated bit-vector solver such as Boolector or Bitwuzla
	 in the disposable container environment.
2. Enumerate fixed circuit topologies rather than asking Z3 to synthesize all
	 constants and masks simultaneously.
3. Use CEGIS: solve on concrete inputs, request a full-domain counterexample,
	 add it, and repeat.
4. Require a solver proof of 32-bit equivalence before testing performance.
5. Explore richer state representations than one fixed XOR mask, for example a
	 two-component affine/XOR encoding that can still be updated cheaply.

### P0: Remove at least 14 gather groups with a factorized lookup

Expected value: high, but only if the selection cost is much lower than a full
binary `vselect` tree.

Constraints:

- Save at least 112 load slots.
- Add at most about 12 flow slots unless another flow optimization is found.
- Avoid increasing the VALU/ALU combined floor above the hash savings.

Promising variants:

- Pair interpolation with MADD leaves and no long final flow chain.
- A two-stage lookup where a small high-bit selection chooses a contiguous
	table block and low bits are handled algebraically.
- Select only groups whose dependency timing fills existing load bubbles,
	rather than selecting a fixed fraction uniformly.
- Co-design the lookup with a nine-operation hash so the freed VALU capacity is
	spent on load reduction.

### P1: Build a clean local virtual-register compiler

Do this independently in the workspace rather than transplanting an external
implementation.

Required invariants:

- Lane-precise reads and writes for `load_offset` and scalarized vector ops.
- Read-before-write semantics for same-cycle/in-place operations.
- Explicit ownership transfer for in-place scalar and vector writes.
- No freeing a physical block until the final reader of the owning version.
- Full operation-count completion checks.
- Offline live-interval validation against 1536 words.

This compiler direction is now integrated in the 983 workspace implementation;
future changes must preserve the invariants above.

### P1: Shorten the final lookup tail

The final table path has a long MADD/vselect chain and accounts for the last
roughly 80 flow-dominated cycles. Revisit it only after reducing the resource
floor; tail scheduling alone cannot overcome the current 972-cycle bound.

Potential experiments:

- Interleave lookup reductions by bit level across all contexts.
- Use a mixed flow/MADD tree chosen from actual per-cycle engine slack.
- Allocate final table versions only after the first wave frees its contexts.
- Solve just the final SSA subgraph with CP-SAT after physical anti-dependencies
	have been removed.

### P2: CP-SAT on a reduced SSA graph

Do not retry whole-program CP-SAT on the physical graph. Instead:

1. Produce a valid greedy SSA schedule and register allocation.
2. Select windows around load-saturated and final-flow regions.
3. Freeze operations outside the window.
4. Optimize the window with capacity, precedence, and live-register bounds.

### Low-priority directions

- Full depth-5/depth-6 static selection without a new factorization.
- More scheduler weight sweeps on the physical graph.
- Early final-table materialization with long-lived table vectors.
- Global fixed-XOR hash encodings using only simple masks.
- Absolute address recurrence that serializes deep rounds.

These have already produced regressions or mathematical impossibility results.

## 9. Scratch and dependency lessons

- Scratch capacity, not only total operation count, controls useful
	parallelism.
- A dead write can act as a lifetime fence. Removing it may expose a much
	longer WAR interval.
- Table setup timing matters less than table-register lifetime.
- Aggregate roofline improvement does not guarantee a shorter schedule; the
	dependency shape must still overlap load, VALU, ALU, and flow work.
- Full SSA is too large, while fixed physical allocation has too many
	anti-dependencies. Selective renaming plus explicit liveness is required.
- Any dynamic allocator must distinguish ownership from address mapping. A
	mapped address can still be freed incorrectly if ownership is not transferred
	to the newest in-place version.

## 10. Disposable artifacts and source of truth

Not part of the submission:

- `/tmp/vliw-1063`: public comparison clone plus disposable experiments.
- `/tmp/ssa_schedule_current.py`: incomplete SSA conversion research script.
- `/tmp/vliw-1063/search_support.py`: reproducible experiment configuration.
- `/tmp/vliw-1063/best_1006_config.py`: prior 1006 checkpoint configuration.
- `/tmp/vliw-1063/random_schedule_search.py`: broad joint scheduler search.
- `/tmp/vliw-1063/local_schedule_descent.py`: single-coordinate descent.
- `/tmp/vliw-1063/local_random_1004.py`: local multi-coordinate search.
- `/tmp/vliw-1063/search_depth4_masks.py`: final lookup-mask search.
- `/tmp/vliw-1063/search_reverse_subsets.py`: per-group bit-order search.
- `/tmp/vliw-1063/physical_reschedule.py` and `cp_sat_physical.py`: physical
	DAG diagnostics; neither belongs in submitted runtime code.
- Any solver outputs or telemetry under `/tmp`.

These are not runtime dependencies and must not be included in a submission.

Current source of truth:

- `perf_takehome.py`: integrated 982-cycle implementation.
- `OPTIMIZATION.md`: this research/state log, including 982 validation.
- `tests/`: unchanged upstream frozen tests.

## 11. Validation commands

Focused iteration:

```bash
cd /home/yanfwang/workspace/original_performance_takehome
/bin/python3 perf_takehome.py Tests.test_kernel_cycles
```

Final official validation:

```bash
cd /home/yanfwang/workspace/original_performance_takehome
/bin/python3 tests/submission_tests.py
git diff --exit-code origin/main -- tests/ problem.py
git diff --check
```

Static diagnostics:

```bash
/bin/python3 - <<'PY'
from collections import Counter
from perf_takehome import KernelBuilder
from problem import SLOT_LIMITS

kb = KernelBuilder()
kb.build_kernel(10, 2047, 256, 16)
slots = Counter()
for bundle in kb.instrs:
		for engine, entries in bundle.items():
				slots[engine] += len(entries)
print("cycles", len(kb.instrs))
print("slots", dict(slots))
print(
		"floors",
		{
				engine: (slots[engine] + SLOT_LIMITS[engine] - 1) // SLOT_LIMITS[engine]
				for engine in ("load", "valu", "alu", "flow", "store")
		},
)
PY
```

The scheduler's normal build output reports peak scratch usage. The old
diagnostic attempted to read `kb.scratch_ptr`, which does not exist for the
dynamic allocator and has been removed from this command.

## 12. Promotion checklist

The integrated 982 checkpoint satisfies all of the following:

1. The scheduler emitted every operation and did not print `STUCK`.
2. The focused frozen-shape test is correct and strictly faster.
3. `tests/submission_tests.py` passes all correctness and speed tests.
4. `git diff --exit-code origin/main -- tests/ problem.py` is empty.
5. Scratch use is at most 1536 words for every generated schedule.
6. No runtime dependency on Z3, OR-Tools, or external repositories.
7. The cycle result is reproduced from a fresh process, not only from a cached
	 builder or an instrumented simulator.
