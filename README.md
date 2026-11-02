# Borrowglass

An OS-free Lisp capability engine and binary target for blind security assessment.
Repository: https://github.com/Sgt-Hwang-6974/Borrowglass
License: Mozilla Public License 2.0 (MPL-2.0); see LICENSE.

Borrowglass targets a custom non-POSIX RISC-V microkernel on StarFive
VisionFive 2 (RV64GC) for the live Black Hat Asia 2027 Arsenal station demonstration.
The engine in `src/*.lisp` is OS-free Common Lisp with zero external dependencies.
The v0.1.0 prototype includes a thin portable Common Lisp CLI review harness,
`run.lisp`, tested under 64-bit Common Lisp using `sbcl --script run.lisp`, so
reviewers and offline investigators can run it locally without board hardware.
No package manager, network service, database installation or Quicklisp is needed.
Run from the release directory:

```sh
sbcl --dynamic-space-size 512 --script run.lisp selftest
sbcl --dynamic-space-size 512 --script run.lisp demo --fixture-seed 1101
sbcl --dynamic-space-size 512 --script run.lisp program examples/ledger.bg
sbcl --dynamic-space-size 512 --script run.lisp program examples/arithmetic.bg
sbcl --dynamic-space-size 512 --script run.lisp replay transcript.bin
sbcl --dynamic-space-size 512 --script run.lisp serve-stdio --fixture-seed 1101 --max-operations 1024
```

Successful commands finish with `BORROWGLASS-COMPLETE command=... status=OK`.
Unhandled serious conditions produce a structured error and exit status 1.
The binary standard-I/O service writes its human-readable completion record and
initial capability material to standard error, keeping standard output binary.
The program runner resolves example paths relative to its own release directory.

## The tenant experiment

There is one relation, identifier 7, with three rows and four unsigned 32-bit
columns. Column zero is the immutable tenant identifier. Row zero belongs to A,
row one to B, and row two to C. A receives privileges 0x0F; B receives 0x01.
Each capability binds a predicate over the tenant column. Requests must authorize
the target column and every predicate column. A successful write returns a new
slice; the table commits that returned slice after checking the immutable tenant
identifier and the proposed row policy. Refusal and filtered results commit nothing.

For a live challenge, `demo --entropy-file PATH` consumes exactly 96 fresh,
cryptographically random bytes supplied by the trusted operator, then creates
a fresh issuer key and protected target in memory. The operator must protect
and remove that external input; the release never stores live entropy or keys. It prints A and B bearer capabilities, a public salt,
and a digest of C's protected row. The issuer key and protected row are never
printed. The target digest binds all four words, including the tenant identifier.
Participants may inspect the source and send requests, but cannot replace the
running executable or inspect its private initialization during the challenge.

For published examples, `demo --fixture-seed 1101` uses public deterministic
initialization and a fixed clock. Output explicitly includes `[FIXTURE-MODE]`.
Fixture target values and keys are public test data. They provide reproducibility,
not secrecy. Arbitrary `.bg` examples run separately from the live ledger.

## Verification and threat model

Borrowglass includes a Lisp implementation whose semantic reference has machine-checked proofs of selected region and linear-capability, binary token-codec, bounded predicate-parser, and evaluator invariants in an unpublished verification suite. Private differential tests compare the separate Lisp implementation with that reference on a recorded finite corpus; they do not prove universal equivalence or establish that the Lisp implementation satisfies every theorem. Other behavior remains unverified work in progress, and module-, function-, and transition-level coverage is intentionally undisclosed. The tenant challenge trusts the host, runtime, issuer key, clock and operator initialization; participants control submitted requests, not the running executable or private initialization. No verified compiler, machine-code correspondence, bare-metal or microkernel verification or execution in v0.1.0, collector-free Common Lisp review runtime, full-stack verification, PostgreSQL completeness or production readiness is claimed.

See docs/USAGE.md for binary layouts and review-host adapters, and
docs/EXPERIMENTS.md for replay, participant budgets and finding adjudication.
