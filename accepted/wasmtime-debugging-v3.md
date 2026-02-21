# RFC: Wasmtime "Debug-Main" Environment

## Summary

This RFC proposes a modification to the [existing RFC
consensus](wasmtime-debugging.md) for *debug components*. In order to
present a more incremental path toward implementation with usable
milestones along the way, it proposes to split the "Debug Adapter
Components" end-goal into two halves, corresponding to the low-level
debug APIs that adapter components consume and high-level debug APIs
that they provide, and build a host for a WIT world that has only the
former first. We call this the "debug main" world. In this
environment, a debug-main component can implement any behavior it
likes on top of the low-level APIs: for example, provide an
implementation of the GDB-stub protocol or a TUI or REPL
user-interface.

## Background: Debug Adapter Components

The [original debugging RFC](wasmtime-debugging.md) described a design
based on *Debug Adapter Components*. These components would interface
with the rest of the system at two ends:

- By importing a "low level" interface providing access to host
  debugging powers at the semantic level of the Wasm virtual
  machine. For example, when attached to a debuggee component, this
  interface would allow pausing and resuming execution, setting
  breakpoints, and examining and modifying the state of Wasm locals
  and the operand stack within each Wasm stack frame and the content
  of any Wasm-level storage (memory, table, global, GC object).
  
- By exporting a "high level" interface that implements debugging
  primitives at the source language level defined by the adapter. This
  level of abstraction has many of the same kinds of primitives, such
  as execution pause and resume, breakpoints, and state observation
  and mutation; but it is meant to work at the source level
  instead. So, for example, breakpoint locations would be denoted in
  terms of line numbers in specific source files, and state
  exploration would present a view of source-level local and global
  variables and any objects or in-memory values accessible via those
  roots.
  
  Providing this high-level interface would necessarily entail parsing
  metadata produced by the source toolchain: e.g., DWARF debug info
  produced by LLVM-based Wasm producers, or in-guest dynamic metadata
  in a dynamic language interpreter Wasm module, or any other custom
  format defined by a given language implementation.
  
  This high-level interface would then be "multiplexed" by some part
  of the Wasmtime debugging infrastructure that would present a
  unified view for the user of a component, possibly composed of
  sub-components each with their own debug adapter.
  
  The high-level interface would correspond more or less to the Debug
  Adapter Protocol's level of abstraction, which is a pure
  source-level view where the UI essentially directly queries the DAP
  provider for locals and their values, and provides uninterpreted/raw
  expressions to parse and evaluate.
  
The intent of this design is to foster an ecosystem of adapters that
correspond to many various kinds of source languages that target Wasm,
providing a universal and unified debugging experience; and to make it
easier to innovate or customize debugging, rather than sliding into a
DWARF-based monoculture by default, which is especially unsuitable
when trying to bridge between interpreted guest languages and compiled
ones as they call across component boundaries.

## Implementation Costs and Difficulties

While this is a laudable goal, we have found that the ambitious size
of the original debugging RFC's proposals has led to a lack of usable
debugging "in the meantime". In particular, to build the full vision,
one needs to build out the "top half" of the debugger -- the part that
understand source-language semantics -- either fully custom, or by
taking apart the pieces of an existing industrial-strength debugger
such as LLDB and then putting them back together in a heavily adapted
way.

Neither of these options is a small project. Needless to say, a "fully
custom" debug adapter for e.g. DWARF-based source languages entails
not only parsing DWARF, but building a framework that understands
enough of the type system and expression syntax of the source language
to provide useful debugging. And then, on top of that, either building
a new framework or replicating the existing framework for custom
pretty-printers, which have their own ecosystem of pluggable Python
modules for gdb/LLDB in the wild for many different projects already
(including, e.g., the Rust standard library). On the other side,
taking pieces of LLDB and putting them together in a novel way is an
enormous undertaking as well, possibly even more difficult as it
requires understanding a large new codebase and potentially
refactoring large parts of it.

This difficult work is necessary under the original plan because the
original plan contains all semantic levels of the debugging stack up
to the "UI view" -- source-level variables and data structures --
within its umbrella. This is a multi-engineer-year project and,
bluntly stated, we cannot afford nor justify that investment.

## An Alternative: Leveraging Existing Debuggers

The major obvious alternative to writing a new debugger "top half" is
to use an existing one, with some interface it already has to talk to
the execution platform. Remote debugging is a well-established
workflow (from the embedded ecosystem and elsewhere) and the de-facto
standard protocol for *low-level* (machine-level) debugging control is
the "gdbstub" protocol, originally defined by GDB and now supported by
LLDB as well.

Our initial debugging SIG discussions dismissed gdbstub for the basic
reason that the protocol does not map cleanly to concepts at the Wasm
VM level: for example, multiple memories, the hidden/protected control
stack, the operand stack, and the like. The need to be "Wasm-native"
was one of two factors that pushed us to DAP and the high-level
interface, along with the desire to abstract across many different
source languages and compiled plus interpreted code.

However, in the meantime, LLDB has [added support for debugging a Wasm
target via an extended gdbstub
protocol](https://github.com/llvm/llvm-project/issues/150449), V8 has
support for this gdbstub protocol
([implementation](https://github.com/llvm/llvm-project/issues/150449)),
and discussion in the LLVM issue indicates JSC is adding support as
well. Thus, a de-facto standard that is *workable* for compiled code
with DWARF metadata has solidified. The gdbstub protocol extensions
include new packet types that deal explicitly with Wasm local and
operand-stack values and the protected Wasm call stack.

We have the "bottom half" of guest-debugging support in Wasmtime
essentially complete now: there are host APIs to set breakpoints,
single-step and continue, receive debug events, and introspect state
(including all frames' PCs, locals and stack values) at every step,
with perfect fidelity. For a usable experience on a pragmatic
timeline, it is desirable to connect this "bottom half" to a gdbstub
implementation. This provides a high-quality debug experience at least
for compiled guest languages, without requiring us to implement a full
"top half" on the scale of LLDB.

However, simply building a gdbstub server as a hardcoded unit of
functionality in (e.g.) the Wasmtime CLI would possibly be a mistake:
it would exert considerable pressure on any new languages targeting
Wasm to adopt the DWARF monoculture, and would pressure them away from
using features such as native Wasm GC that cannot be debugged with the
extended gdbstub protocol or LLDB's C-level understanding of the
target machine. Ideally we would build a kind of extensible
*framework* that could support new kinds of debuggers in the future,
while still giving us a practical path to debugging in the near term.

## A Compromise: "Debug Main"

We thus propose defining a new WIT world for Wasm components that act
as debug providers, in the vision of the original RFC, but without the
"top half" exported API. Instead, the world consists of the union of:

- The low-level API to introspect and control a debuggee
  component/module, and
- The WASI CLI world.

One could then implement any number of top halves in this world:

- A gdbstub provider, listening on a TCP socket (via `wasi-sockets`);
- A DAP provider, listening on a TCP socket or HTTP endpoing (via
  `wasi-http`);
- A text-mode REPL or a TUI providing a direct machine-monitor
  debugging experience, similar to old-school `DEBUG.COM`;
- Any other kind of introspection tool or interface.

In other words, this is more flexible because it is not prescriptive
about the other end of the adapter, but it does not preclude building
the DAP-based adapter ecosystem *on top* of the new world.

The implementation of this debug-main world has been prototyped in
[this
branch](https://github.com/cfallin/wasmtime/tree/a-whole-wide-world-for-debuggers-to-play-in),
with the WIT
[here](https://github.com/cfallin/wasmtime/blob/a-whole-wide-world-for-debuggers-to-play-in/crates/debugger/wit/world.wit),
and appears to work decently. It has exposed some limitations of the
existing WIT language (for example, every Wasm value must be a
resource, because one cannot borrow or clone a variant that has
resources in some leaves) but nothing that cannot be solved.

This RFC is not seeking consensus on the exact API design, but rather
providing this as an example of the kinds of interfaces that will be
available. In general, we want to expose the full introspection
ability that the native Rust API of Wasmtime provides; so, for
example, though the prototype does not have it yet, we should have the
ability to examine GC types and GC objects, and create new ones; make
arbitrary function calls; inject exceptions, early returns or
different return values; and the like. In fullness of time, this could
grow to become a full reflection API, usable by some components to
dynamically launch and inspect other components, but such a use-case
is also beyond the initial scope of this work.

## Alternatives

- We could stick to our plan as described in the initial RFC, and
  develop a full debug-adapter ecosystem. This would likely take a
  year or more of sustained effort, which (i) is more than we can
  afford at our current engineering staffing levels, and (ii) would
  lead to increasingly unbearable pressure caused by the lack of a
  good debugging experience.
  
- We could lean into the "attach native gdb/LLDB" workflow
  instead. This has been seen to have fairly sizeable issues with
  source-level state fidelity, and has a history of bugs due to a
  complex and poorly-maintained DWARF translator. We have already done
  the work to build a perfect-fidelity and reliable
  instrumentation-based "bottom half" for the debugger, and we should
  not abandon the effort to use it.
  
- We could build a direct gdbstub implementation in native host code,
  rather than as a Wasm component. This has been tempting at various
  points, but we believe shortcuts too much of the
  adaptability/extensibility goals and cuts off future possibilities
  too soon.

## Acknowledgments

Thanks to Nick Fitzgerald and Alex Crichton for extensive discussions
on this topic.
