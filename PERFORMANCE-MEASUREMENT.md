# Performance Measurement

How to measure the speed of a library function so that the number
describes what a consumer gets, and two rules for hot-path code that the
same incidents produced.

Status: **proposed [2026-11-26], not adopted.** From corvus v1.2.1
(OldCrow/corvus#117, open). corvus meets every rule below on that branch.
No other repo has been audited against it, so by this repo's own rule
("no rule without a follower") it binds nobody until the user adopts it
and each repo records where it stands. Not yet named in any repo's
`AGENTS.md`.

It adds to each repo's own rules for a quiet machine and a Release build;
it replaces none of them.

## Why this exists

libstats adopted corvus and its scalar Gaussian quantile, one `erfcinv`
call, went from 72 to 106 ns on the Zen 4 machine. The search for the
cause took one day and produced three wrong conclusions before the right
ones, each published or acted on before it was checked:

1. An experiment showed a one-value call could cost 29 ns instead of 65,
   and a patch release was decided on it. The experiment compiled its own
   copy of the kernel and passed raw pointers; the shipped library did
   neither.
2. The fix was built into the library, gained 3 to 5 ns, and the
   experiment was written off as a harness artefact. That was a guess. The
   effect was real and the fix was half of it.
3. A before-and-after comparison restored a header from a saved copy. The
   copy kept its old timestamp, the library did not rebuild, and both
   programs measured the same code.

What was actually there, none of it in what the kernel computes:

- **Calls that could not overlap.** A loop of independent one-value calls
  ran at the kernel's latency, about twice its throughput, because a load
  in each call could not be forwarded from the stores it overlapped and
  waited for them to retire. One such load was in the library's masked
  tail. The other came from passing a 16-byte `std::span` by value, which
  under the Windows x64 calling convention is a 16-byte load of two 8-byte
  stores.
- **Cost that depended on the process's memory.** A masked load with an
  all-false mask reads nothing, so it never maps its page, and it took a
  slow path for as long as the page stayed unmapped: about 90 ns per
  vector, four times the kernel around it. Whether a program was slow
  depended on where its linker put one constant. Identical machine code
  read 3.6 or 14.6 ns per element.
- **A harness that was not the library.** A test program that compiled
  the kernel header itself got a different layout from a program that
  linked the library, and read fast where every real consumer read slow.

No accuracy gate and no throughput bench in either repo could have caught
any of the three.

## Rules for measuring

1. **Measure the shipped artefact from a program shaped like a
   consumer.** A claim about a library's speed comes from a program that
   links the library and nothing of its internals. A program that compiles
   a kernel header itself measures its own build: the linker folds
   identical functions across the two and places constants differently.
   Such a program is fine for comparing variants against each other; it
   says nothing about the library until the library reproduces it.
2. **Compare before and after from two separate build trees.** Check out
   each version in its own directory and build both. Do not swap a file in
   one tree: a restored copy can keep its timestamp and skip the rebuild.
   State something that shows the two builds differ, such as the object's
   size.
3. **Measure in the caller's shape.** A function consumers call one value
   at a time is timed that way too: the value passed in and the result
   returned, in a loop, beside the array-to-array figure. The two can
   differ by a factor of two, and only the first is what the consumer
   pays. Say which shape a figure is.
4. **Measure cold as well as warm.** Time in a fresh process, and where
   the operating system allows it, again after its pages have been
   unmapped (on Windows, trimming the working set). A figure that changes
   between the two states is a defect to find, not noise to average.
5. **Reproduce through rule 1 before acting.** A surprising gain or loss
   found in an experiment is rebuilt into the library and measured there
   before it is posted, planned on or released. A figure from a machine in
   use is called indicative wherever it is written.
6. **Say whether the mechanism is demonstrated or inferred.** "Fixed"
   means an experiment turned the effect on and off. A cause without that
   experiment is a guess and is written as one. When a result contradicts
   an earlier one, find what differs between the two before deciding which
   was wrong.

## Rules for hot-path code

These are narrower. The first applies to any C++ that may be built for
Windows; the second to SIMD kernels with AVX-512 targets.

7. **A non-inlined function on a per-value path takes pointers and
   scalars, not a small aggregate by value.** `std::span`,
   `std::string_view` and any struct over 8 bytes travel, under the
   Windows x64 convention, as a pointer to a copy. A wrapper that forwards
   one copies it with a single wide load of what the caller wrote as
   separate narrow stores, and that load cannot be forwarded. The public
   signature can keep the aggregate; the function behind it should not.
8. **A rare-path constant stays behind a branch.** In a SIMD kernel, an
   unconditional select of a special value can compile to a masked load of
   the constant from memory, executed on every call with a mask that is
   almost always empty. Test once whether any lane needs the special case
   and skip the block.

## Applying it

- **corvus:** the rules are in `docs/NUMERICAL-DOCTRINE.md` (Kernel
  construction) and `docs/PERFORMANCE.md` §6. `tests/bench_span1.cpp`
  measures both caller shapes; `tests/test_masked_const.cpp` compares the
  trimmed and mapped states; `tests/test_span1.cpp` holds the results
  equal.
- **libstats:** not audited. It passes spans through its own layers and
  is validated on the same Windows machine, so rules 3, 4 and 7 are the
  ones to check first.
- **Every other repo:** not audited.

## What is not known

- The masked-load cost is demonstrated on one CPU and one operating
  system (Zen 4, Windows 11). Its mechanism, a slow path for suppressed
  faults, is inferred from on/off experiments; no hardware counters were
  taken.
- The calling-convention half of rule 7 does not apply under the System V
  convention, which passes a 16-byte aggregate in two registers. Whether
  Linux and macOS builds gain anything from it is unmeasured.
- Rule 4 has a ready method on Windows only.
