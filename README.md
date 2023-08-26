# GSoC23 - Adapting IR Load Semantics to Freeze All or Freeze Only Uninitialized Data

This repository contains documentation in support of GSoC23 project "Adapting IR
Load Semantics to Freeze All or Freeze Only Uninitialized Data".

## LLVM Patches

* [\[llvm\]\[MemoryBuiltins\] Add alloca support to getInitialValueOfAllocation](https://reviews.llvm.org/D155773)
* [WIP: \[Bitcode\]\[C-API\]\[IR\] Introduce bitcode load version 2 and freeze_bits metadata](https://reviews.llvm.org/D158342)
* [WIP: \[UpdateTestChecks\] Add update test check support for freeze_bits metadata](https://reviews.llvm.org/D158343)
* [WIP: \[UpdateTestChecks\] Add update test check support for fpmath metadata](https://reviews.llvm.org/D158344)
* [WIP: \[clang\]\[llvm\]\[test\] Update tests to support freeze_bits metadata](https://reviews.llvm.org/D158345)
* [WIP: \[llvm\]\[MemoryBuiltins\] Add initialization category to getInitialValueOfAllocation](https://reviews.llvm.org/D158352)
* [WIP: \[mem2reg\] Refactor load of uninitialized memory to poison semantics](https://reviews.llvm.org/D158353)

## GitHub

* [llvm-project clone used for in progress patch development](https://github.com/jmciver/llvm-project/tree/development/jmciver/freeze-bits/squash)
* [alive2 clone used for freeze_bits support](https://github.com/jmciver/alive2/tree/development/jmciver/freezing-load)

## Discourse

The request for comment leading to the development of this project and follow on
discussions can be found at the following link:

* [\[RFC\] Load Instruction: Uninitialized Memory Semantics ](https://discourse.llvm.org/t/rfc-load-instruction-uninitialized-memory-semantics/67481/1)

## Implementation

The project implementation is based on the following steps:

| Task Group | Status |
| ---------- | ------ |
| Bitcode load version 2 | <span style="color:green">Completed</span> |
| Add freeze metadata support | <span style="color:green">Completed</span> |
| BitCode auto-update support | <span style="color:green">Completed</span> |
| Update C-API | <span style="color:green">Completed</span> |
| Update regression tests for freeze pre-optimization refactor | <span style="color:green">Completed</span> |
| Update MemoryBuiltins interface for freeze metadata | <span style="color:green">Completed</span> |
| Refactor optimizations for freeze metadata | <span style="color:yellow">In progress</span>
| Performance metric validation of freeze | <span style="color:red">Outstanding</span> |
| Add uninit_is_nondet metadata support | <span style="color:orange">TBD</span> |
| Refactor optimizations for uninit_is_nondet | <span style="color:orange">TBD</span> |


### Bitcode

A deterministic method is required to differentiate between bitcode load
instruction pre-uninitialized undef and poison semantics in order to provide an
automatic in place upgrade strategy. The cleanest and historically supported
methodology is to add a new instruction version operational code and provide a
method of auto-upgrade in the AutoUpgrade translational object.

The augmentation requires the reader to support both load v1 and load v2 while
the writer only supports load v2. This many to one mapping is enabled by the
addition of the metadata freeze_bits, which provides backwards compatibility
as well as supports bit-field type constructs in the environment where
uninitialized memory loads default to poison.

Work is completed on adding the new load instruction operational codes: 21 for
non-atomic and 53 for atomic.

### Metadata freeze

A technical issue was discovered during development where TableGen output for
freeze metadata conflicted with the existing freeze instruction token. To
quickly resolve this issue an adapted metadata name, freeze_bits, was used for
further development.

Support for metadata freeze_bits is completed both in the IR and bitcode data
structures. IR data structures are automatically handled using TableGen
declarations in Attributes.td while bitcode is handled using C-style
enumerations in LLVMBitCodes.h. As the bitcode reader does not make direct use
of the IRBuilder a method `llvm::UpgradeLoadInstruction` was added in migrate
load v1 by adding the attribute !freeze_bits thus converting the v1 to v2.

### C-API Support for freeze_bits

Construction of load instruction using the LLVM C-API is handled by unwrapping
and wrapping of an IRBuilder object. Prior to refactoring for IR load version 2,
an upgrade to the load instruction LLVMBuildLoad had occurred to support opaque
pointers resulting in a new function LLVMBuildLoad2.

To provide backwards compatibility, LLVMBuildLoad2 now calls the default
CreateLoad constructor with isFreezing enabled. A new function is added,
LLVMBuildLoad3, which calls CreateLoad with isFreezing set to false. The
function LLVMBuildLoad2 has been marked with the deprecate attribute using the
macro `LLVM_ATTRIBUTE_C_DEPRECATED`. All unittest and end-to-end tests have been
updated to use LLVMBuildLoad3.

### MemoryBuiltins getInitialValueOfAllocation

The API change is concentrated around the function
`getInitialValueOfAllocation`, which has been refactored to return a
`std::pair`` of an enumeration and an IR constant. The enumeration designates
how the constant was derived, and the constant initially conformed to current
behavior with the caveat that undef will become poison.

The following is the getInitialValueOfAlloction function declaration currently
used in main:

```cpp
Constant *llvm::getInitialValueOfAllocation(
  const Value *V,
  const TargetLibraryInfo *TLI,
  Type *Ty)
```

Which is now refactored to:

```cpp
std::pair<InitializationCategory, Constant *>
  llvm::getInitialValueOfAllocation(
    const Value *V,
    const TargetLibraryInfo *TLI,
    Type *Ty,
    const LoadInst *Inst = nullptr)
```

Support for alloca instructions has be added. This allows for a consistent query
to be used for both stack and heap. Calls to `getInitialValueOfAllocation` using
an alloca initially resulted in undef to match existing semantics but have since
been refactored to return poison. A load instruction parameter has been added
for the emission of FreezePoison when the alloca or allocation function does not
initialize and freeze_bits attached to the load instruction being surveyed for
transformation.

#### Enumerations

The following enumerations are supported:

* FreezePoison
* Constant
* Unknown

#### FreezePoison

This enumerator is returned when the load is accompanied by freeze_bits and
classification of the allocator is determined to be in an
uninitialized state. The returned constant is `std::nullptr``.

The MemoryBuiltins API is used by clients that remove loads and stores. The
reason for the null of arbitrary type is it is differentiated from zero. If zero
were used optimizations like dead store elimination (DSE) would remove the store
0.

#### Constant

This enumerator is returned when the load is determined to be of an address that
was allocated using:

* `calloc` or compliant new derivative (MSVC etc.)
* `malloc` in the absence of freeze_bits

In the first case the returned constant will be zero. This is in keeping with
current behavior where initialized `AllocFnKind::Zeroed` results in a zero
constant of arbitrary type. In the second case the return constant will be
poison.

#### Unknown

This enumerator is returned when getInitialValueOfAllocation is unable to
determine the classification of the allocation operations. No constant object is
produced and thus the returned pointer will be a nullptr. This provides backward
compatibility with the current API.

### Optimization Refactoring

Optimization support for freeze_bits has been completed for mem2reg, SROA,
InstCombine, and general analysis ValueTracking.

A modified clone of alive2 has been created to support freeze_bits analysis.

## Future Work

The remainder of optimization updates for freeze_bits support is on going. Once
this is completed patches for the remaining optimizations will be submitted. A
series of patches relating to each optimization's regression tests will be
submitted. Finally, performance analysis will be completed and results posted on
Discourse. 

If performance is determined adequate work will then begin to mainline
the patches. In the event performance or semantics issues are identified work
will begin on uninit_is_nondet metadata.
