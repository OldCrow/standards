# Numerical Kernel Promotion

What to do when a consumer library (libstats, libhmm) finds a numerical
method that is as accurate as, and faster than, what it has: fix the
problem where it was found, then promote the method to corvus, the fleet's
special-function engine, so that one copy serves every consumer.

Status: **adopted [user, 2026-10-04]**, from libstats v2.5.0 (the von
Mises rewrite, below). First applications: OldCrow/corvus#44 (the von
Mises quadrature machinery; fixed-cost incomplete gamma/beta as a
research question) and OldCrow/corvus#45 (span `expm1`, `log1pmx`). Not
yet named in any repo's `AGENTS.md`.

## Why this exists

corvus's own doctrine is the standard way serious special-function
libraries are built: split the domain into regions and evaluate each with a
fixed-cost method (a polynomial, an asymptotic series, a fixed quadrature),
carrying extra precision through the hard regions. Consumers that follow the
same doctrine will sometimes find a method corvus does not have. The question
this answers is what happens to it afterwards, so that the fleet does not end
up with two diverging copies, or with the better method stranded in one
library.

The incident: libstats v2.4.2 fixed the von Mises CDF and quantile for
accuracy and made the quantile 52–136× slower than v2.4.1. The fix that
recovered the speed (to 5–7× v2.4.1, with the CDF faster than v2.4.1 at
κ ≥ 2) replaced a Bessel series and an adaptive quadrature with fixed
Gauss–Legendre rules and Watson-lemma series, region by region, on a
cancellation-free integral. The reusable parts of that method (verified
quadrature tables and their generator, compensated sums, Watson-series
helpers) belong to no distribution, and some may beat corvus's continued
fractions for the incomplete gamma and beta.

## Rules

1. **Fix it where you find it.** A consumer does not wait for corvus to
   fix its own defect or regression. The fix ships in the consumer's next
   release.
2. **Write it to be moved.** A kernel that may be promoted is:
   - self-contained, with its inputs, outputs and error bound stated;
   - documented with its regime map: which method serves which region, and
     the evidence for each boundary;
   - free of hand-pasted constants. Every table is produced by a committed
     generator that also has a `--check` mode, and the source marks the
     generated block. *Incident: the first von Mises table had 4 of 168
     Gauss–Legendre node pairs off by up to 1.1e-16, which defeated their
     two-double precision; tests could not see it, and a regenerating
     check could.*
   - validated against an oracle independent of its own method (for von
     Mises: the Bessel series at 360 digits, not mpmath's adaptive
     quadrature, which failed in the deep tail at large κ);
   - guarded by tests shown to fail without the kernel's key step.
3. **File it with corvus.** Open a corvus issue that names the kernel, its
   region map, its accuracy and cost evidence, and the consumer commit. It
   needs no milestone to be filed.
4. **corvus decides, by its own doctrine.** For a function corvus already
   covers, the candidate competes on accuracy first, then on speed on every
   tier, then on maintenance cost. A faster method usually wins only some
   regions, so the usual outcome is a better region map, not a replacement.
   corvus records the methods considered and why each region's winner won.
5. **One copy afterwards.** Once corvus adopts the method, the consumer
   deletes its copy and calls corvus, as part of that consumer's
   corvus-adoption work. Until then, the consumer's copy keeps the
   signatures and constant tables corvus will take, so the swap is
   mechanical.
6. **What stays local.** A kernel for a function that is not a special
   function in corvus's sense (a distribution's CDF built on its own
   integral, such as von Mises) stays in the consumer. Only its reusable
   machinery is promoted.

## Applying it

- A consumer's `AGENTS.md` names any kernel it holds under rule 5, with its
  corvus issue.
- corvus's `AGENTS.md` names the issues it has accepted under rule 3, and
  where their region-map records live.
