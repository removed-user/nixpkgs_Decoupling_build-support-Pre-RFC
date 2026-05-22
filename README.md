# nixpkgs_Decoupling_build-support-Pre-RFC

Pre-RFC Idea for decoupling build support from pkgs, into a set of higher-order functions.


# RFC: Decoupling `build-support` into a Deferred Higher-Order Function Library (`PkgsLib`)

## Abstract
This RFC proposes a structural and architectural shift in how Nixpkgs manages its build orchestration tools. Currently, `build-support` utilities (e.g., `buildGoModule`, `buildRustPackage`, `fetchFromGitHub`) live within the `pkgs` attribute set, in a sense - treating package blueprints as if they're packages themselves...

This doesn't make any sense at all; and seems **literally** like an architectural relic, that came into existence because everyone agreed they don't belong in lib... 

But "not belonging in lib" doesn't mean they automatically belong in pkgs either. Realistically - build-support functions... function as if they are simply a "second lib", which "happens to require" a pkgs arguement. 

So, maybe they should be treated like exactly that. 

I propose refactoring `build-support` into a deferred, higher-order function library. eg:  `PkgsLib`... or similar — and moving it outside the core package instantiation scope. 
 
By passing `pkgs` as a lazy input argument through a multi-stage evaluation pipeline, we get strict conceptual alignment, eliminate core infinite recursion risks, and optimize evaluation latency.

(Full disclosure, I'm no nix-expert and asked an AI to help put my thoughts/idea's, into a cohesive argument, which is here): feel free to add to the case for a change like this + inform me of anything incorrect/stupid, and how it could be made less stupid.



https://github.com/removed-user/nixpkgs_Decoupling_build-support-Pre-RFC

Also, feel free to check out/add to this repo.

### The repo where I'm working on extensions to a few build-support functions, which might be pulled into/become future proof of concept repo, when finished
https://github.com/removed-user/Nix_Extra-Builders

---

## 1. Motivation & Conceptual Misalignment

In the current Nixpkgs architecture, the boundary between `lib` and `pkgs` is defined operationalized by system dependence:
* **`lib`** contains pure, system-agnostic utility functions (logic).
* **`pkgs`** contains platform-dependent derivations (software).

Because `build-support` functions require compilers, linkers, and fetchers, they are deeply system-dependent and are forced to reside inside `pkgs`. However, this introduces a deep conceptual flaw: **builders are not packages; they are higher-order functions that output package recipes.** 

Treating builders as standard attributes inside a mutually recursive package set creates two core problems:
1. **The Bootstrap Catch-22:** Fetchers and basic builders frequently cause infinite recursion loops if a downstream package override accidentally alters an upstream dependency used by the builder itself.
2. **Monolithic Evaluation Overhead:** Because builders are bound directly inside the massive `pkgs` fixpoint loop, decoupling structural build logic from concrete software compilation is incredibly difficult, impeding tree-wide optimization and modularity.

---

## 2. Proposed Architecture

I propose introducing an explicit/linear three-stage evaluation pipeline that separates pure tool bootstrapping, deferred build logic, and final package materialization.





### Stage 0: The Immutable Bootstrap Layer
A minimal, raw set of primitive derivations (e.g., standard `stdenv`, `bash`, `curl`) is evaluated immediately. This layer is entirely locked down and cannot accept downstream overrides.

### Stage 1: The `PkgsLib` Scope
`build-support` is converted into a library of partially applied functions or deferred modules. 
* Primitive actions (like `fetchgit`) pull their execution tools strictly from **Stage 0**.
* High-level abstractions (like `buildGoModule`) accept the final package set (`finalPkgs`) as a lazy, late-bound argument to reference target compilers.

### Stage 2: Final Resolution Loop (`pkgs`)
Using an open fixpoint (`lib.fix`), the final user-facing package set is materialized by feeding the instantiated `pkgs` back into `PkgsLib`.

---

### Architectural Deep-Dive: Mechanics and Benefits

#### 1. Passing `pkgs` as a Lazy Input Argument
* **The Mechanism:** Instead of nesting builders directly inside the global package scope, `PkgsLib` is structured as a deferred function: `makePkgsLib = finalPkgs: { ... }`.
* **The Impact:** This introduces a "late binding" phase. The blueprint logic for a builder sits entirely dormant in memory. It does not look for compilers, libraries, or dependencies until a fully instantiated `pkgs` object is explicitly passed to it at the final step of evaluation.

#### 2. Achieving Strict Conceptual Alignment
* **The Mechanism:** `build-support` is removed from the `pkgs` attribute set and structurally relocated into its own functional layer.
* **The Impact:** This fixes a fundamental philosophical category error. In current Nixpkgs, `pkgs.buildGoModule` is structurally handled the exact same way as `pkgs.hello` (a discrete derivation). Relocating it to a dedicated library ensures the code structure reflects reality: builders are an engineering *library of functions*, not compiled software artifacts.

#### 3. Eliminating Core Infinite Recursion Risks
* **The Mechanism:** The multi-stage pipeline establishes a strict, one-way dependency boundary between the bootstrap layer (`Stage 0`) and the finalized packages (`Stage 2`).
* **The Impact:** This completely isolates a major vector for evaluation failures. 
  * *The Status Quo:* If a user writes an override to patch `curl` globally, `fetchgit` immediately tries to use that patched `curl`. If building that patched `curl` requires fetching a source code repository via `fetchgit`, evaluation crashes with an infinite recursion error.
  * *The New Pipeline:* The `fetchgit` builder inside `Stage 1` is strictly pinned to `bootstrapPkgs.curl` from `Stage 0`. Downstream package overrides in `Stage 2` cannot cross the boundary to reach backward into `Stage 0`. `fetchgit` safely fetches the source using the immutable bootstrap tools, breaking the recursive loop entirely.

#### 4. Optimizing Evaluation Latency
* **The Mechanism:** This leverages Nix’s native **lazy evaluation** model by separating structural abstractions from concrete values.
* **The Impact:** In standard Nixpkgs, because builders are tightly coupled to the global recursive fixpoint, the evaluator must frequently wade through complex package dependency trees just to resolve a builder's environment. Moving builders to a pure function layer creates a radical shortcut: the evaluator processes the builder logic instantaneously, deferring heavy package compilation logic until the exact moment a specific attribute path is built.

---

## 3. Conceptual Implementation Example

```nix
let
  lib = import <nixpkgs/lib>;

  # STAGE 0: Isolated Bootstrap Tools
  ## Evaluates immediately. These are pinned and completely immune to downstream overrides,
  ## breaking the primary vector for infinite recursion in standard fetchers.
  ## evaluating only up to the point that we have a barebones coreutils+minimalCurl+stdenv (bound to system), basically running the stage0/hex0/mesC up to coreutils build process

  bootstrapPkgs = {
    stdenv = { /* primitive stdenv */ };
    curl = { /* primitive curl */ };
    coreutils = { /* primitive coreutils */ };
  };

  # STAGE 1: PkgsLib (Deferred Higher-Order Builders Library)
  # A decoupled logic layer. It takes finalPkgs as a late-bound argument.
  makePkgsLib = finalPkgs: {
    # Fetchers pin strictly to Stage 0 tools to ensure absolute purity and stability
    fetchgit = { url, sha256 }: bootstrapPkgs.stdenv.mkDerivation {
      name = "source";
      buildInputs = [ bootstrapPkgs.curl ];
      # ... fetch logic
    };

    # Ecosystem builders defer compiler selection to finalPkgs, allowing for dynamic overrides
    buildGoModule = { src, ... }@args: bootstrapPkgs.stdenv.mkDerivation (args // {
      buildInputs = [ finalPkgs.go ] ++ (args.buildInputs or []);
    });
  };

  # STAGE 2: Reconstructed Materialized Packages
  # The final open fixpoint loop where the user-facing package set is constructed.
  pkgs = lib.fix (finalPkgs: 
    let 
      pkgsLib = makePkgsLib finalPkgs;
    in {
      # Compilers live safely inside the final tier
      go = bootstrapPkgs.stdenv.mkDerivation { /* go compiler recipe */ };

      # Packages call upon the deferred library, passing inputs down cleanly
      myGoApp = pkgsLib.buildGoModule {
        pname = "app";
        version = "1.0";
        src = pkgsLib.fetchgit { url = "https://github.com"; sha256 = "..."; };
      };
    }
  );
in
  pkgs.myGoApp
```

---

## 4. Key Advantages

### A. Total Elimination of Circular Bootstrapping
By forcing a strict downward data flow (Stage 0 $\rightarrow$ Stage 1 $\rightarrow$ Stage 2), a primitive tool like `fetchgit` will never query `finalPkgs` for its environment. This effectively patches an entire class of infinite recursion bugs that plague complex package overrides.

### B. Perfect Laziness and Memory Optimization
Because the structural rules of `PkgsLib` are separated from package instantiation, Nix's lazy evaluation engine shines. No package derivation or compiler blueprint is evaluated until its specific attribute path is explicitly invoked. 

### C. Tree-wide Customization without Global Rebuilds
Downstream users can safely swap out compilers or flags inside `finalPkgs` without accidentally mutating the underlying fetching or building framework mechanics, enabling clean ecosystem testing.

---

## 5. Community Proofs of Concept: dream2nix, devenv, flake-parts

Because modifying the main Nixpkgs repository requires navigating massive legacy compatibility constraints, developers have built greenfield projects to prove that your proposed layout works flawlessly at scale.

### dream2nix:
 This project completely reimagines packaging by turning builders (buildGoModule, buildRustPackage) into modular, discrete plug-ins. 
It handles package constraints almost exactly like this PkgsLib pipeline: it passes an instantiated pkgs attribute set downstream into structural module blueprints.

### devenv and flake-parts

These systems heavily rely on specialArgs and deferred module functions to safely pass things like the system tuple (x86_64-linux) or pkgs downstream only when needed.


## 6. Drawbacks and Implementation Challenges

* **Refactoring Effort:** Transitioning the entire Nixpkgs tree away from `pkgs.buildGoModule` to a decoupled structure represents a colossal breaking change. 
* **Compatibility Layer Required:** A backwards-compatible shim would be mandatory, mapping `pkgs.buildGoModule` back to `pkgsLib.buildGoModule` so that existing user expressions do not break.
* **Evaluation Performance Tuning:** While lazy evaluation is protected, managing large-scale fixpoint injection across 80,000+ packages would have to be benchmarked to ensure the Nix evaluator does not suffer from excessive memory overhead.
