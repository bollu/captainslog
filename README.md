## 2022 August 8

- Dev meeting: next thursday, 8 am pacific time (4 pm BST).

#### Target triple

- Cmake can find target triple: https://github.com/google/libcxx/blob/master/cmake/Modules/GetTriple.cmake
- LLVM knows everything about a triple: https://llvm.org/doxygen/group__LLVMCTarget.html
    - LLVMGetTargetFromTriple: const char * → LLVMTargetRef
    - LLVMCreateTargetMachine: LLVMTargetRef → LLVMTargetMachineRef
    - LLVMCreateTargetDataLayout: LLVMTargetMachineRef → LLVMTargetDataRef
    - LLVMIntPtrType: LLVMTargetDataRef → LLVMTypeRef
    - Triple::getOs: llvm::Triple → OS [can be used for our `#if WINDOWS`]

#### Leanc
  - Must add a `--print-triple` to get target triple info?
  - How does rust do this?

#### lean_inline.cpp
  - Had to manually copy from `lean_inlines.h` and replace `static inline` with `extrn "C" LEANSHARED`.
  - More sensible distribution story?

#### Action points
  1. Send PR for LLVM bindings. [`LLVMBindings.lean`. Merge this]
  2. Add lake support for LLVM IR via facets?
  3. Don't we only need `lean_inlines.cpp`? Why do we need the others? We can link `libleanshared` separately
     at link-time, and only use `lean_inlnes.cpp` during LLVM module optmisation time.
  4. We need three tools: `llvm-link`, `opt`, `llc`. (this is *Critical*).
  5. `lean_system_platform_nbits` to get the number of bits in a pointer.
  6. Bootstrapping? Solve the `i32/i64` difference between the two platforms. Use a type alias that is later specialized?
     (This is *Critical*).
  7. New IR: memory layout.
  8. Have `alwaysinline` and `musttailcall` in `EmitLLVM`.
  9. Split the compilation pipeline in `.lean` to `.olean`, followed by `.olean` to `.ir`,and then to `.bc/.llvm/.c/.wasm`.
     (Ask in the lean4-dev, Mac would know where we would need to change in `lake`.)
  10. `ccache`? Maintain cache ourselves (?)

#### Notes by `mac` on writing a plugin for `lake` for new backend

Here are some notes on a how a new backend (and a separate C backend) would be implemented in Lake.

## Separate C Backend

If the C backend is separated from the `olean`/`ilean` build (and is thus able
to be built separately), one should ideally remove the whole `isLeanOnly`
configuration and replace it with just a separate build step.

This would entail removing the `isLeanOnly` option from `PackageConfig` and
`Package`, the C build from `Module.buildUnlessUpToDate`, and the C related
code from `Module.recBuildLean`. Then, one would add a separate build function
as below for the C code.

## New Backend

To add a new build step from the `olean/ilean`, add a new facet ala the current
`o` facet (and its build function `buildLeanO`) that waits for the the module
build and then builds the desired file. For example:

```lean
def Module.recBuildLeanLl (self : Module) : IndexBuildM (BuildJob FilePath) := do
  buildLeanLl self.name.toString self.llFile (← self.leanBin.fetch) self.llArgs

def Module.oFacetConfig : ModuleFacetConfig llFacet :=
  mkFacetJobConfig Module.recBuildLeanLl
```

