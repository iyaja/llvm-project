=============================================
Building a JIT: Per-function Lazy Compilation
=============================================

.. contents::
   :local:

**This tutorial is under active development. It is incomplete and details may
change frequently.** Nonetheless we invite you to try it out as it stands, and
we welcome any feedback.

Chapter 3 Introduction
======================

**Warning: This text is currently out of date due to ORC API updates.**

**The example code has been updated and can be used. The text will be updated
once the API churn dies down.**

Welcome to Chapter 3 of the "Building an ORC-based JIT in LLVM" tutorial. This
chapter discusses lazy JITing and shows you how to enable it by adding an ORC
``CompileOnDemand`` layer the JIT from `Chapter 2 <BuildingAJIT2.html>`_.

Lazy Compilation
================

When we add a module to the KaleidoscopeJIT class from Chapter 2 it is
immediately optimized, compiled and linked for us by the ``IRTransformLayer``,
``IRCompileLayer`` and ``RTDyldObjectLinkingLayer`` respectively. This scheme, where all the
work to make a Module executable is done up front, is simple to understand and
its performance characteristics are easy to reason about. However, it will lead
to very high startup times if the amount of code to be compiled is large, and
may also do a lot of unnecessary compilation if only a few compiled functions
are ever called at runtime. A truly "just-in-time" compiler should allow us to
defer the compilation of any given function until the moment that function is
first called, improving launch times and eliminating redundant work. In fact,
the ORC APIs provide us with a layer to lazily compile LLVM IR:
``CompileOnDemandLayer``.

The ``CompileOnDemandLayer`` class conforms to the layer interface described in
Chapter 2, but its ``addModule`` method behaves quite differently from the layers
we have seen so far: rather than doing any work up front, it just scans the
Modules being added and arranges for each function in them to be compiled the
first time it is called. To do this, the ``CompileOnDemandLayer`` creates two small
utilities for each function that it scans: a *stub* and a *compile
callback*. The stub is a pair of a function pointer (which will be pointed at
the function's implementation once the function has been compiled) and an
indirect jump through the pointer. By fixing the address of the indirect jump
for the lifetime of the program we can give the function a permanent "effective
address", one that can be safely used for indirection and function pointer
comparison even if the function's implementation is never compiled, or if it is
compiled more than once (due to, for example, recompiling the function at a
higher optimization level) and changes address. The second utility, the compile
callback, represents a re-entry point from the program into the compiler that
will trigger compilation and then execution of a function. By initializing the
function's stub to point at the function's compile callback, we enable lazy
compilation: The first attempted call to the function will follow the function
pointer and trigger the compile callback instead. The compile callback will
compile the function, update the function pointer for the stub, then execute
the function. On all subsequent calls to the function, the function pointer
will point at the already-compiled function, so there is no further overhead
from the compiler. We will look at this process in more detail in the next
chapter of this tutorial, but for now we'll trust the CompileOnDemandLayer to
set all the stubs and callbacks up for us. All we need to do is to add the
``CompileOnDemandLayer`` to the top of our stack and we'll get the benefits of
lazy compilation. We just need a few changes to the source:

.. code-block:: c++

  ...
  #include "llvm/ExecutionEngine/Orc/CompileOnDemandLayer.h"
  #include "llvm/ExecutionEngine/Orc/EPCIndirectionUtils.h"
  ...

  ...
  class KaleidoscopeJIT {
  private:
    std::unique_ptr<TargetMachine> TM;
    const DataLayout DL;
    RTDyldObjectLinkingLayer ObjectLayer;
    IRCompileLayer<decltype(ObjectLayer), SimpleCompiler> CompileLayer;

    using OptimizeFunction =
        std::function<std::shared_ptr<Module>(std::shared_ptr<Module>)>;

    IRTransformLayer<decltype(CompileLayer), OptimizeFunction> OptimizeLayer;

    std::unique_ptr<JITCompileCallbackManager> CompileCallbackManager;
    CompileOnDemandLayer<decltype(OptimizeLayer)> CODLayer;

First we need to include the CompileOnDemandLayer.h header, then add two new
members: a ``std::unique_ptr<EPCIndirectionUtils>`` and a ``CompileOnDemandLayer``,
to our class. The ``EPCIndirectionUtils`` member is used by the ``CompileOnDemandLayer```
to create the compile callback needed for each function.

.. code-block:: c++

  public:
    KaleidoscopeJIT(std::unique_ptr<ExecutorProcessControl> EPC,
                    std::unique_ptr<ExecutionSession> ES,
                    std::unique_ptr<EPCIndirectionUtils> EPCIU,
                    JITTargetMachineBuilder JTMB, DataLayout DL)
        : EPC(std::move(EPC)), ES(std::move(ES)), EPCIU(std::move(EPCIU)),
          DL(std::move(DL)), Mangle(*this->ES, this->DL),
          ObjectLayer(*this->ES,
                      []() { return std::make_unique<SectionMemoryManager>(); }),
          CompileLayer(*this->ES, ObjectLayer,
                      std::make_unique<ConcurrentIRCompiler>(std::move(JTMB))),
          OptimizeLayer(*this->ES, CompileLayer, optimizeModule),
          CODLayer(*this->ES, OptimizeLayer,
                  this->EPCIU->getLazyCallThroughManager(),
                  [this] { return this->EPCIU->createIndirectStubsManager(); }),
          MainJD(this->ES->createBareJITDylib("<main>")) {
      MainJD.addGenerator(
          cantFail(DynamicLibrarySearchGenerator::GetForCurrentProcess(
              DL.getGlobalPrefix())));
    }

Once we've have to updated our constructor to initialize the new ``EPCIndirectionUtils`` member, we can construct our ``CompileOnDemandLayer``. Following the pattern from
previous layers we start by passing a reference to the next layer down in our
stack -- the OptimizeLayer. Next we need to supply a 'partitioning function':
when a not-yet-compiled function is called, the CompileOnDemandLayer will call
this function to ask us what we would like to compile. At a minimum we need to
compile the function being called (given by the argument to the partitioning
function), but we could also request that the CompileOnDemandLayer compile other
functions that are unconditionally called (or highly likely to be called) from
the function being called. For KaleidoscopeJIT we'll keep it simple and just
request compilation of the function that was called. Next we pass a reference to
our CompileCallbackManager. Finally, we need to supply an "indirect stubs
manager builder": a utility function that constructs ``IndirectStubManagers``, which
are in turn used to build the stubs for the functions in each module. The
``CompileOnDemandLayer`` will call the indirect stub manager builder once for each
call to ``addModule``, and use the resulting indirect stubs manager to create
stubs for all functions in all modules in the set. If/when the module set is
removed from the JIT the indirect stubs manager will be deleted, freeing any
memory allocated to the stubs. We supply this function by using the
``createLocalIndirectStubsManagerBuilder`` utility from the ``EPCIndirectionUtils`` member.

With that, our KaleidoscopeJIT implementation now supports a form of lazy compilation where each function we supply is compiled only when first called through the use of function stubs and compile callbacks. There are many ways this can be customized. In the next chapter, we'll look at ``MaterializationUnit``s and the lazy re-exports API that takes laziness a step further by JITing directly from the Kaleidoscope AST, which is perhaps more useful in practice - why do extra work on a function that is never called? Another way to improve performance is through concurrency. If we can compile eagerly on a seperate thread, we can free the main thread executing our code and avoid some of the startup time penalties incurred by using lazy JITing with optimizations. We'll exlpore this idea and other possible solutions to the lazy-eager trade-offs in chapter 5 and beyond.

`Next: Extreme Laziness - Using LazyReexports to JIT from ASTs <BuildingAJIT4.html>`_

Full Code Listing
=================

Here is the complete code listing for our running example with a CompileOnDemand
layer added to enable lazy function-at-a-time compilation. To build this example, use:

.. code-block:: bash

    # Compile
    clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
    # Run
    ./toy

Here is the code:

.. literalinclude:: ../../examples/Kaleidoscope/BuildingAJIT/Chapter3/KaleidoscopeJIT.h
   :language: c++

`Next: Extreme Laziness -- Using Compile Callbacks to JIT directly from ASTs <BuildingAJIT4.html>`_
