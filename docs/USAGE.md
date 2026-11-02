# Borrowglass usage and binary interfaces

Source and releases: https://github.com/Sgt-Hwang-6974/Borrowglass
The MPL-2.0 OS-free engine targets a custom non-POSIX RISC-V microkernel on
StarFive VisionFive 2 (RV64GC) for the live station demonstration. The v0.1.0
portable Common Lisp CLI review harness is tested under 64-bit Common Lisp
with SBCL and lets reviewers work offline without the board.
Start from the release directory with `sbcl --dynamic-space-size 512 --script
run.lisp selftest`. All binary integers below are little-endian. Signed time
fields use Unix seconds and two's-complement signed 64-bit encoding.

## Capability encoding: exactly 384 bytes

| Offset | Bytes | Field |
| --- | --- | --- |
| 0 | 4 | format version, exactly 1 |
| 4 | 8 | issuer key identifier |
| 12 | 8 | subject role identifier |
| 20 | 8 | relation identifier |
| 28 | 8 | column mask |
| 36 | 4 | privileges: SELECT bit 0, UPDATE bit 2, reserved bit 4 clear |
| 40 | 8 | not-before Unix seconds, inclusive |
| 48 | 8 | expiry Unix seconds, exclusive |
| 56 | 8 | revocation epoch |
| 64 | 1 | predicate byte length |
| 65 | 1 | caveat byte length |
| 66 | variable | prefix predicate, then caveats, then zero padding |
| 352 | 32 | HMAC-BLAKE2s-256 over bytes 0 through 351 |

The deferred serial field is zero and consumes no wire bytes. Predicate opcodes
are true=1, false=2, column/literal=3, column/column=4, and=5, or=6, not=7.
Comparison bytes are eq=1, ne=2, lt=3, le=4, gt=5, ge=6. Comparison leaves begin
with opcode, comparison, and two zero reserved bytes; then a 32-bit column index
and either a signed 64-bit literal or another 32-bit column index. Boolean nodes
carry their subtrees immediately in prefix order. Predicate bounds are depth 16,
16 nodes, and 34 parser work steps. There are at most four 16-byte caveats, each
containing a 64-bit narrowing column mask and signed 64-bit maximum expiry.
All remaining bytes before the tag must be zero. A token needs SELECT set even
when it also carries UPDATE. The token is a replayable bearer capability.

## Request framing: exactly 128 bytes per frame

Every frame starts with ASCII BRG1 at 0..3, version 1 at byte 4, kind at 5,
zero flags at 6, and fragment index at 7. Bytes 8..15 hold a grouping identifier;
16..23 are zero; 24..27 hold the payload length; 28..31 are zero. Payload occupies
32..127. Token kind 1 uses indices 0,1,2,3 in order and length 96. Command kind 2
is READ, kind 3 is WRITE; index is 4 and payload length is 24.

The command payload is relation:u64, row:u32, column:u32, replacement:u32,
reserved:u32. READ replacement and all command padding must be zero. Only one
transfer may be pending. Interleaving, duplication, malformed padding, and
out-of-order fragments clear it. Every input frame, including malformed and
truncated frames, counts toward the operation budget.

Response kind 128 uses a 24-byte payload: status:u32 at payload offset 0,
phase:u32 at 4, refusal code:u32 at 8, projected value:u32 at 12, and zeros
thereafter. Status 0 means accepted, 1 refused, 2 filtered, 3 protocol error.
Phase 1 is framing; 2 authentication/admission; 3 codec; 4 static/predicate;
5 runtime/commit. A refusal never returns a projected value.

## Programs and command workflows

`program examples/ledger.bg` executes a slice update in an isolated fixture;
`program examples/arithmetic.bg` exercises signed division. The short source
syntax is `(program (add (i64 19) (i64 23) 22003))`, also supporting sub, mul,
div, mod and scalar literals. Names remain strings. Files are ASCII, at most
16,384 bytes, depth 64 and atom length 128; host reader macros are rejected.
`demo --fixture-seed 1101` shows the public tenant baseline. `serve-stdio --max-operations
1024` processes bounded binary input. Capture stderr separately from responses.
`replay transcript.bin` validates a BGT1 public-fixture transcript through the
same session-frame transition in memory and compares every recorded response byte.

The complete form is `(program MODULE ENTRY ENV TUPLE FUEL)`. MODULE is
`(module NAME REGIONS DECLARATIONS COLS MASK EFFECTIVE POLICY WHERE CLOSED)`;
REGIONS contains `(NAME ID CLASS BYTE-CAP BASE)` entries. CLASS is static, pool,
session, transaction, plan or row. POLICY and WHERE are predicate forms or `none`.
Predicates are `(true)`, `(false)`, `(literal COL CMP I64)`, `(columns COL CMP COL)`,
`(and P P)`, `(or P P)` and `(not P)`; CMP is eq, ne, lt, le, gt or ge.

A function declaration is `(fn NAME REGION-PARAMS PARAMS RETURN-TYPE EFFECTS
ANNOTATIONS BOUND BODY)`. REGION-PARAMS contains `(NAME CLASS MODE)` entries;
PARAMS contains `(NAME TYPE)` entries. Types are i32, i64, u32, u64, bool,
`(words REGION MODE CAP)`, `(bytes REGION MODE CAP)`, `(token KIND REGION)` and
`(result (OK-TYPES...) ALLOW-FILTERED ERROR-CODE)`. MODE is shared, unique or
consume. Token kinds are region-cap, row-borrow, buffer-pin, pool-handle, borrow
and mut-borrow. Constants use `(constant NAME REGION TYPE (BYTE...))`.

Expression forms include `(var NAME TYPE)`, scalar literals, add/sub/mul/div/mod,
`(word32 OP E E)`, `(word64 OP E E)`, `(cmp CMP E E)`, `(boolean OP E E)`,
`(shift DIRECTION E E WIDTH ERROR-CODE)`, `(call NAME (ARGS...))`, and
`(slice-ref E E TOKEN ERROR-CODE)` / `(byte-ref E E TOKEN ERROR-CODE)`.
Statements are let, if, loop, match, return, slice-set, byte-set and primitive.
The ledger example gives a complete slice-set form including both continuations.
`(loop BOUND DEPTH ((NAME TYPE INIT)...) STEP AFTER)` carries typed bindings;
STEP returns continue or exit. `(return MODE (EXPRS...) (TOKENS...) CODE)`
requires CODE for refuse; other modes are ok, filtered, continue and exit.
`(match E ((NAME TYPE)...) YES (ERROR-NAME (PRESERVED...)) NO)` handles results.
Annotations are `(lifetime REGION CLASS MODE)` and `(variant BOUND DEPTH)`.
The six primitive names are bg-arena-allocate, bg-row-reenter, bg-region-open,
bg-row-end, bg-generation-next and bg-token-authenticate.

ENV entries pair a name with a scalar, slice, cap, row-token, scratch or result
value; `examples/ledger.bg` shows word-slice and row-token encodings. An invalid
form yields a structured refusal. The static checker decides type, ownership,
region, earlier-function-call and primitive-module admission after decoding.
User code has no host reader, evaluator, compiler, loader or live table access.

For live initialization, supply `--entropy-file PATH` with exactly 96 fresh
cryptographically random bytes from the trusted environment. No random source
is guessed; absent input refuses. Protect and remove that external file after
initialization. Public fixture seeds are reproducible and provide no secrecy.
The core consumes explicit byte vectors, state and clock callbacks. `run.lisp`
contains file I/O and review-host adapters: SBCL supplies argv, exit status and
binary standard streams. Other Common Lisp hosts may load with `*run-main*` nil,
call `dispatch-command` explicitly, and supply octet standard streams for service;
no cross-implementation execution or board runtime execution is claimed today.
