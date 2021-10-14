# Summary
[summary]: #summary

This RFC proposes to improve control flow integrity for compiled WebAssembly code by utilizing two
technologies from the Arm instruction set architecture - Pointer Authentication and Branch Target
Identification.

# Motivation
[motivation]: #motivation

The [security model of WebAssembly][wasm-security] ensures that Wasm modules execute in a sandboxed
environment isolated from the host runtime. One aspect of that model is that it provides implicit
control flow integrity (CFI) by forcing all function call targets to specify a valid entry in the
function index space, by using a protected call stack that is not affected by buffer overflows in
the module heap, and so on. As a result, in some Wasm applications the runtime is able to execute
untrusted code safely. However, the burden of ensuring that the security properties are upheld is
placed on the compiler to a large extent.

On the other hand, a further aspect of the WebAssembly design is efficient execution (close to
native speed), which leads to a natural tendency towards sophisticated optimizing compilers.
Unfortunately, the additional complexity increases the risk of implementation problems and in
particular compromises of the security properties. For example, Cranelift has been affected by
issues such as [CVE-2021-32629][cve] that could make it possible to access the protected call stack
or memory that is private to the host runtime.

We are trying to tackle the challenge of ensuring compiler correctness with initiatives such as
expanding fuzzing and making it possible to apply formal verification to at least some parts of the
compilation process. However, it is also reasonable to consider a defense in depth strategy and to
evaluate mitigations for potential future issues.

Finally, Wasmtime can be used as a library and in particular embedded into an application that is
implemented in languages that lack some of the hardening provided by Rust such as C and C++. In that
case the compiled WebAssembly code could provide convenient instruction sequences for attacks that
subvert normal control flow and that originate from the embedder's code, even if Cranelift and
Wasmtime themselves lack any defects.

[cve]: https://github.com/bytecodealliance/wasmtime/security/advisories/GHSA-hpqh-2wqx-7qp5
[wasm-security]: https://webassembly.org/docs/security

# Proposal
[proposal]: #proposal

Currently this proposal focuses on the AArch64 execution environment.

## Background

The Pointer Authentication (PAuth) extension to the Arm architecture protects function returns, i.e.
provides back-edge CFI. It is described in section D5.1.5 of
[the Arm Architecture Reference Manual][arm-arm]. Some of the PAuth operations act as `NOP`
instructions when executed by a processor that does not support the extension. Furthermore, a code
generator can use either one of two keys (A and B) for the pointer authentication instructions; the
architecture does not impose any restrictions on any of them, leaving that to the software
environment.

The Branch Target Identification (BTI) extension protects other kinds of indirect branches, that is
provides forward-edge CFI and is described in section D5.4.4. Whether BTI applies to an executable
memory page or not is controlled by a dedicated page attribute. Note that the `BTI` "landing pad"
for indirect branches acts as a `NOP` instruction when the extension is not active (e.g. for
processors that do not support BTI).

Both extensions are applicable only to the AArch64 execution state and are optional, so the usage of
each CFI technique will be controlled by dedicated settings. Wasmtime embedders need to consider a
subtlety - the setting values may happen to be located in memory that could be potentially
accessible to an attacker, so the latter could disable the use of PAuth and BTI in subsequent code
generation. Mitigating this issue is outside the scope of this proposal.

The article [*Code reuse attacks: The compiler story*][code-reuse-attacks] and the whitepaper
[*Pointer Authentication on ARMv8.3*][qualcomm-pauth] provide an introduction to the technologies.

In the Intel® 64 architecture [the Control-Flow Enforcement Technology (CET)][intel-cet] provides
similar capabilities.

[arm-arm]: https://developer.arm.com/documentation/ddi0487/gb/?lang=en
[code-reuse-attacks]: https://community.arm.com/arm-community-blogs/b/tools-software-ides-blog/posts/code-reuse-attacks-the-compiler-story
[intel-cet]: https://www.intel.com/content/www/us/en/developer/articles/technical/technical-look-control-flow-enforcement-technology.html
[qualcomm-pauth]: https://www.qualcomm.com/documents/whitepaper-pointer-authentication-armv83

## Improved back-edge CFI with PAuth

Assuming that the A key is used, the proposed implementation will add the `PACIASP` instruction to
the beginning of every function compiled by Cranelift and will replace the final return with either
the `RETAA` instruction or a combination of `AUTIASP` and `RET`.

In environments that use the DWARF format for unwinding the implementation will be modified to apply
the `DW_CFA_AARCH64_negate_ra_state` operation or an equivalent immediately after the `PACIASP`
instruction.

Those steps will be skipped for simple leaf functions that do not construct frame records on the
stack.

As a conrete example, consider the following function:

```plain
function %f() {
    fn0 = %g()

block0:
    call fn0()
    return
}
```

Without the proposal it will result in the generation of:

```plain
  stp fp, lr, [sp, #-16]!
  mov fp, sp
  ldr x0, 1f
  b 2f
1:
  .byte 0x00, 0x00, 0x00, 0x00
  .byte 0x00, 0x00, 0x00, 0x00
2:
  blr x0
  ldp fp, lr, [sp], #16
  ret
```

And with the proposal:

```plain
  paciasp
  stp fp, lr, [sp, #-16]!
  mov fp, sp
  ldr x0, 1f
  b 2f
1:
  .byte 0x00, 0x00, 0x00, 0x00
  .byte 0x00, 0x00, 0x00, 0x00
2:
  blr x0
  ldp fp, lr, [sp], #16
  retaa
```

Associated AArch64-specific Cranelift settings - the default values are always `false`:
* `has_pauth` - specifies whether the target environment supports PAuth
* `sign_return_address` - the main setting controlling whether the back-edge CFI implementation is
used; results in the generation of operations that act as `NOP` instructions unless `has_pauth` is
also enabled
* `sign_return_address_all` - specifies that all function return addresses will be authenticated,
including the previously mentioned cases that do not need it in principle
* `sign_return_address_with_bkey` - changes the generated instructions to use the B key; note that
this is enforced for any Apple ABI, irrespective of the value of this setting

## Enhanced forward-edge CFI with BTI

The proposed implementation will add the `BTI j` instruction to the beginning of every basic block
that is the target of an indirect branch and that is not a function prologue. Note that in the
AArch64 backend generated function calls always target function prologues and indirect branches that
do not act like function calls appear only in the implementation of the `br_table` IR operation.
On the other hand, function prologues will begin with the `BTI c` instruction, keeping in mind that
Cranelift does not have any special handling of tail calls. If PAuth is used at the same time, then
the initial `PACIASP`/`PACIBSP` operation will act as a landing pad instead.

There is only one associated AArch64-specific Cranelift setting, `use_bti`, which is `false` by
default. Wasmtime will set the respective memory protection attribute for all executable pages if
the WebAssembly module has been compiled with that setting enabled; similarly for the Cranelift JIT.

## CFI improvements to code that is not compiled by Cranelift

Currently the code that is not compiled by Cranelift is in assembly, C, C++, or Rust.

Improving CFI for compiled C, C++, and Rust code with the same technologies is outside the scope of
this proposal, but in general it should be achievable by passing the appropriate parameters to the
respective compiler.

Functions implemented in assembly will get a similar treatment as generated code, i.e. they will
start with the `PACIASP` instruction (and any unwinding directives), assuming that the A key is
used. However, the regular return will be preserved and instead will be preceded by the `AUTIASP`
instruction. The reason is that both `AUTIASP` and `PACIASP` act as `NOP` instructions when executed
by a processor that does not support PAuth, thus making the assembly code generic. Functions that do
not need the pointer authentication operations will start with the `BTI c` instruction instead.

One potential problem in the interaction between code that is compiled by Cranelift and code that is
not is that only one side might have the CFI enhancements. However, this proposal does not have any
ABI implications, so Rust code in the Wasmtime implementation that does not use PAuth and BTI, for
example, would be able to call functions compiled by Cranelift without any issues and vice versa.
The reason is that it is the responsibility of the callee to ensure that PAuth is used correctly,
while everything is transparent to the caller. As for BTI, if an executable memory page does not
have the respective attribute set, then the extension does not have any effect, except for
introducing extra `NOP` instructions, irrespective of how the code has been reached (e.g. via a
branch from a page with BTI protections enabled); similarly for branches out of the unprotected
page. The major exception that is relevant to Wasmtime is unwinding, but there should be no issues
as long as the abovementioned DWARF operation is used and the system unwinder is recent.

Future work that is beyond what this proposal presents may introduce further hardening that
necessitates ABI changes, e.g. by being based on
[the proposed PAuth ABI extension to ELF][pauth-abi] or something similar.

[pauth-abi]: https://github.com/ARM-software/abi-aa/blob/2021Q3/pauthabielf64/pauthabielf64.rst

### Fiber implementation in Wasmtime

The fiber implementation in Wasmtime consists of a significant amount of assembly code that will
receive the treatment described in the previous section, as an initial implementation. However, the
fiber switching code saves the values of all callee-saved registers on the stack, i.e. memory that
is potentially accessible to an adversary. Some of those values could be code addresses that would
be used by indirect branches, so a complete CFI implementation will verify the integrity of the
saved state with the `PACGA` instruction.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Since the existing implementation already uses the standard back-edge CFI techniques that are
preferred in the absence of special hardware support (i.e. a separate protected stack that is not
used for buffers that could be accessed out of bounds), the alternative is not to implement the
proposal, so the rationale is based mainly on the overhead being insignificant. In terms of code
size the impact of the back-edge CFI improvements is 1 or 2 additional instructions per function.

The [Clang CFI design][clang-cfi-design] provides an idea for an alternative implementation of the
forward-edge CFI mechanism that is enabled by BTI. It involves instrumenting every indirect branch
to check if its destination is permitted. While the overhead of this approach can be reduced by
using efficient data structures for the destination address lookup and optionally limiting the
checks only to indirect function calls, it is still significantly larger than the worst-case BTI
overhead of one instruction per basic block per function. On the other hand, it does not require any
special hardware support, so it could be applied to all supported platforms.

[clang-cfi-design]: https://clang.llvm.org/docs/ControlFlowIntegrityDesign.html

# Open questions
[open-questions]: #open-questions

- What is the performance overhead of the proposal?
