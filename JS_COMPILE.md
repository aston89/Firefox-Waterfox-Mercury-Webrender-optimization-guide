# JIT 

## JavaScript compilation in Firefox is not a single “compile the page” operation.
SpiderMonkey treats each top-level script and each function as a separate compilation unit. Large web applications can therefore contain thousands of independently executable scripts and functions, many of which may never be executed during a particular page load.

This is one of the reasons modern Firefox does **not** immediately compile every byte of JavaScript into highly optimized native code.

The simplified pipeline looks like this:

```text
JavaScript source
       ↓
 parsing / syntax processing
       ↓
 bytecode / Stencil
       ↓
 C++ interpreter
       ↓
 Baseline Interpreter + Inline Caches
       ↓
 Baseline JIT
       ↓
 WarpMonkey
       ↓
 optimized native machine code
```

### Lazy parsing: don't compile what nobody executes

Firefox normally uses lazy parsing for functions.

The parser can recognize a function and perform the required syntax/error checks without immediately generating the full bytecode for its body. The function is only fully compiled later if execution actually reaches it.

This saves both CPU time and memory because large web applications often ship considerably more JavaScript than they execute during their initial page load.

When a previously lazy function is executed, SpiderMonkey performs **delazification** and generates the missing bytecode for that function.

For very large applications, changing delazification settings can therefore have a surprisingly large effect on startup CPU usage and memory consumption.

### Tiering: code gets hotter before it gets expensive

SpiderMonkey uses multiple execution tiers.

The Baseline Interpreter is a fast execution layer that also gathers information through Inline Caches. The Baseline JIT then turns the bytecode into native code with relatively inexpensive compilation.

The highest optimization tier is WarpMonkey.

Warp uses the bytecode together with information collected by the Baseline execution tiers to build an intermediate representation (MIR), optimize that graph, lower it to LIR, perform register allocation and finally generate native machine code.

The important point is that **each tier costs more to compile but can produce faster code**.

This creates a trade-off:

```text
low threshold
    ↓
more code promoted earlier
    ↓
higher compilation cost
    ↓
more native code + more temporary compiler memory
    ↓
potentially lower execution overhead
```

while:

```text
high threshold
    ↓
less code promoted
    ↓
less compilation work
    ↓
lower memory pressure
    ↓
some code remains in cheaper execution tiers for longer
```

This is why aggressive warm-up settings can make a browser feel faster on repeatedly executed JavaScript while also increasing CPU activity and temporary memory usage.

### Why compilation can use vastly more memory than the JavaScript itself

A JavaScript file is not compiled directly from source text into machine code in one step.

Warp first builds an intermediate representation of the program and then performs multiple optimization passes before generating the final machine code.

Conceptually:

```text
bytecode
   ↓
Warp snapshot
   ↓
MIR graph
   ↓
optimization passes
   ↓
LIR
   ↓
register allocation
   ↓
code generation
   ↓
native machine code
```

The intermediate structures can be considerably larger than the original bytecode.

Inlining can increase this effect further.

When a hot call site is considered suitable for inlining, the callee can effectively become part of the caller's optimization graph:

```text
caller
  │
  ├── operation
  ├── call → function A
  │            ↓
  │         inline A
  │            ↓
  │       larger MIR graph
  │
  └── more optimization opportunities
```

This can improve execution speed by eliminating call overhead and exposing more code to optimization, but it also makes the compiler's temporary working set larger.

This is why an aggressive JIT configuration can produce a pattern such as:

```text
400–500 MB
     ↓
new code path becomes hot
     ↓
Warp compilation starts
     ↓
temporary MIR/LIR/codegen structures
     ↓
RAM spikes by several GB
     ↓
compilation finishes
     ↓
temporary structures are released
     ↓
RAM falls again
```

A large temporary spike therefore does **not** necessarily mean that Firefox permanently decided to keep that amount of memory.

### Off-thread compilation: keeping the browser responsive

Warp compilation is typically performed off the main thread.

The main thread gathers the information needed for compilation while background compiler threads construct and optimize the MIR/LIR representation and generate native code.

This is important for large single-page applications because a large compilation job can consume substantial CPU and memory without necessarily blocking the UI for its entire duration.

Firefox also supports off-main-thread Baseline compilation. Depending on the selected strategy, Baseline compilation can be performed eagerly when useful cached information is available, and/or on demand when execution requires it.

This creates an important distinction:

```text
CPU parallelism
    ≠
JavaScript execution becoming fully multithreaded
```

A page may still execute its normal JavaScript on its main execution context while Firefox performs parts of parsing and JIT compilation on background threads.

### Trial inlining: optimization based on hot call sites

Trial inlining is deliberately more local than “compile the whole application harder”.

SpiderMonkey tracks execution information for individual scripts and Inline Cache call sites. A call site must become sufficiently hot before it can become a candidate for trial inlining.

This means that an aggressive `inliningEntryThreshold` does not simply instruct Warp to inline everything.

Instead, it changes how quickly individual call sites become eligible:

```text
rare call
   ↓
ignored

warm call
   ↓
candidate

very hot call
   ↓
trial inlining
   ↓
specialized optimization
```

The trade-off is that lowering the threshold causes more call sites to reach this stage, including call sites that may later prove not to benefit enough from inlining.

For this reason, extremely low values can increase compiler activity and memory consumption dramatically without producing a measurable improvement in page responsiveness.

### Bailouts and recompilation

Warp optimizations are speculative.

The optimizer makes assumptions based on the types, shapes and execution patterns observed during previous execution. If optimized code encounters a situation that does not match those assumptions, SpiderMonkey can perform a bailout and resume execution through a lower tier.

Frequent bailouts can eventually cause the optimized script to be invalidated.

This is another reason why “more aggressive compilation” is not automatically better:

```text
more aggressive assumptions
        ↓
more optimization opportunities
        ↓
but also
        ↓
more opportunities for speculation to become invalid
        ↓
bailout / invalidation / recompilation
```

The best JIT configuration is therefore not necessarily the one that compiles the largest possible amount of JavaScript. It is the one that spends compilation time and memory on code that remains hot enough to justify the investment.

### JitHints: learning from previous execution

SpiderMonkey also has JitHints, which use information from previous execution to accelerate tiering decisions for scripts encountered again.

This makes the system less dependent on blindly waiting for every warm-up threshold from zero.

The general idea is:

```text
previous execution
       ↓
"this function became hot"
       ↓
hint stored
       ↓
next encounter
       ↓
earlier compilation can be attempted
```

This is one reason modern Firefox can become faster after repeated use of the same web application without requiring the user to manually lower every JIT threshold.

### The important takeaway

A browser JIT is not trying to answer:

> "How do I compile the entire SPA as aggressively as possible?"

It is trying to answer:

> "Which pieces of this program are hot enough to justify spending more CPU time, compiler memory and generated-code memory on them?"

Aggressive tuning changes that decision boundary.

Lower thresholds, broader inlining limits and larger Ion admission limits can move substantially more code into the expensive optimization pipeline.

That can improve throughput for workloads that repeatedly execute the same hot paths.

It can also produce very large **temporary compilation spikes**, especially on large SPAs.

The goal of JIT tuning is therefore not maximum compilation.

It is **maximum useful compilation**.
