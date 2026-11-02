# Reproducible blind assessment

Repository: https://github.com/Sgt-Hwang-6974/Borrowglass
The MPL-2.0 OS-free engine targets a custom non-POSIX RISC-V microkernel on
StarFive VisionFive 2 (RV64GC) for the live station. Review v0.1.0 offline
using its portable Common Lisp CLI harness, tested under 64-bit Common Lisp
with `sbcl --script run.lisp`, without board hardware.
Read README.md's verification and threat-model statement before interpreting
results. A failed attack is an observation within a budget, not a security proof.

## Participant task and fixed budget

Participants receive the entire released executable source, an A administrator
capability, a B read-only capability, a public salt and C target digest. Give each
participant the same time, number of submitted frames, machine limits and initial
briefing. Record source-review time separately from execution time. Classify a
finding as unauthorized B mutation, unauthorized C disclosure, unhandled runtime
condition, or deterministic state corruption. Preserve the exact input bytes and
release digest for each result. A grouping identifier is never authority.

The operator controls initialization, clock, issuer key and executable. Participants
control request bytes and separately loaded example programs. Editing a local
copy of the source is useful for analysis, but success against an edited server
is not a challenge finding. Root access to the challenge process is outside the
experiment. Keep the property-level coverage map permanently confidential.
Do not provide adaptive hints classifying individual modules, branches or findings.
Confidential adjudication checks each finding against the predeclared scope.

## Public fixtures and live initialization

`demo --fixture-seed 1101` derives repeatable fixture bytes and fixes Unix time at
1700000000. The output label `[FIXTURE-MODE]` identifies this public test context.
A fixture token has no authority in a freshly initialized live process. Live
issuer keys and protected values remain in RAM. This release stores neither a
live key nor a secret boundary nonce. Live CLI initialization requires 96 fresh
cryptographically random bytes via `--entropy-file PATH`; the trusted operator
protects and removes that external input. Do not claim reproducibility of a live
finding from a public seed: it requires the operator's retained process context
and corresponding clock inputs.

The BGT1 transcript container begins with 32 bytes: ASCII BGT1, version:u32=1,
fixture seed:u64, clock:u64=1700000000, input byte count:u32, response byte
count:u32, then input bytes and expected response bytes. The replay command bounds
file size at 8 MiB, reconstructs the public fixture, and invokes the same
`session-frame` transition as the standard-I/O service, in memory. A changed response is
a replay failure. Binary transcripts contain public fixture requests and projected
responses, with no live key or unrecovered protected row.

## External assessment and interpretation

Participants bring their own auditors, independent input generators and AI tools
against the 128-byte BRG1 interface and .bg capability engine. The release supplies
selftest and replay commands. The self-test checks a real A write followed by a
read, B's allowed read and denied writes, C confidentiality, operator recovery,
and expiry/floor boundaries. Broader independent inputs remain valuable: a finite
corpus cannot establish universal equivalence, compiler correctness or absence of
bugs. Record timing, resource use and discovered defects separately from coverage
guesses. Absence of findings does not establish a causal security benefit.
The live station target is planned; v0.1.0 review-harness results do not establish
execution or verification of a board runtime, compiler or complete stack.
