# Source Text Phase Imports

## Status

Champion(s): Luca Casonato, Guy Bedford

Stage: 0

## Problem Statement

This proposal seeks to solve the _static worker_ module analysis problem for
JavaScript, through defining suitable phase imports for Source Text Module.

## Motivation

Improving runtime and tooling support for workers will enable faster JavaScript
applications.

The `new Worker()` constructor pattern that is currently relied on for these
workflows suffers from a number of analysis issues:

1. It always takes an arbitrary expression to locate the worker URL. The worker
   is not just a _static import_, like normal ESM imports.
2. The string passed to `new Worker(url)` doesn't support module resolution
   rules. Because relative URLs are resolved baseURL-relative, users usually
   need to rely on a resolution function first, such as an out-of-band
   configuration or expressions involving `import.meta.resolve()` or
   `import.meta.url`. There are many different patterns here and no single
   standard approache employed by developers, further exacerbating any analysis
   attempts as per problem (1).

Usage examples:

```js
// Common case, but requires very non-trivial static analysis
const url = new URL("./my_worker.js", import.meta.url);
const worker = new Worker(url, { type: "module" });

const url = import.meta.resolve("./my_worker.js");
const worker = new Worker(url, { type: "module" });

// This can quickly turn into near impossible static analysis for most tools
function createWorker(specifier) {
  return new Worker(url, { type: "module" });
}

const url = new URL("./my_worker.js", import.meta.url);
const processor = createWorker(url);
```

The result is a situation in which is is difficult to reliably statically
analyze which modules are loaded in workers, causing issues for runtimes and
tools:

* `new Worker` is not ergonomic for developers to use when authoring
  applications and especially when authoring libraries.
* Tools have difficulty both analyzing and bundling applications that use
  workers, resulting in less usage and limited compatibility for shared
  libraries to support workers.

A better language primitive for worker loading can resolve these static analysis
improving theses workflows as well as their analysis and build tooling.

## Proposal

By defining a new phase for ECMAScript module records, it is possible to 
import a handle to a module statically, and pass this handle to the worker:

```js
import somephase myModule from "./my-module.js";

// `{ type: 'module' }` can be inferred since myModule is a module object
const worker = new Worker(myModule);
```

This technique solves analysis problems (1) and (2) for worker imports in
improving the runtime worker ergonomics - supporting static worker references,
while resolving as module-relative via the normal module resolution rules with
all resolution features supported.

In addition, the improved static analysis makes it possible for tools to analyze
the worker creation much more easily, to determine that a static `myModule`
handle is being passed directly to `new Worker`. Bundling can be performed by
replacing the `./my-module.js` phase import with a phase import to the fully
optimized worker chunk to load.

This new phase would then also lay the ground work for the future proposals for
module harmony proposals in defining the new phases.

Since phases also support a dynamic import form, we would also get the dynamic
variant:

```js
const workerModule = await import.somephase('./worker.js');
new Worker(workerModule);
```

## Background

With the recently introduced [Source Phase Imports][], it is now possible to
define import phases that exist prior to the full linking and execution of the
module graph.

These phases form the basis of a number of new proposals for modules including
module declarations, module expressions, deferred module imports, virtualization
through loaders and new worker semantics.

In order to develop these new proposals in a way that aligns with phased
imports,Currently this phase is not yet specified for ECMAScript modules
themselves, as we do not have a definition of this object yet for JavaScript.

One of the driving specification constraints for these objects is also how they
behave when transferred between agents and workers, which we would argue forms
a critical design space for these features, which is why this proposal is seen
as the most suitable "next step" in the larger layering efforts.

## Design Questions

### Module Instance v Module Source

There is an outstanding design question as to whether this phase should be an
instance or source phase for the module.

The instance phase represents an entire graph of modules, while the source phase
represents just a specific module source without its linking being defined.

The benefit of the instance phase being that because it represents the graph, it
might be more amenable to preloading.

Work to determine which phase to specify is ongoing within this proposal's
current stage. It may yet specify one or the other, or it may even specify
_both_ a source and an instance to allow handling some of the deeper tradeoffs
and layering questions.

### Transferability

The phase defined for ECMAScript modules may or may not be a transferable phase,
via `postMessage` into the worker. The transerability question is also separate
to the question of supporting `new Worker(phase)` invocation, which can be
supported for a given phase as separate to the concept of transder.

The source phase is currently transferable, while the instance phase may have
more difficulties in transferability due to having to handle edge cases around
module identity for dependency transfer.

## Layering

### Source Phase Imports

This proposal is designed to work in conjunction with [Source Phase Imports][],
whether or not it directly specifies a source phase for ECMAScript modules.

### Import Attributes

The [Import Attributes Proposal][] provides a way to pass attributes to the
module loader. These attributes are used during source loading and resolution.
Because th module source and module instance have already gone through this
process, they are already _attributes-influenced_ by the time their handle is
obtained.

When passing an module instance or module source object to a dynamic `import()`
or `new Worker`, any additional `with` attributes would therefore be unsupported
- and setting attributes would throw an error.

### Deferred imports

The [Deferred Imports][] proposal provides a way to defer the synchronous
evaluation of a module until it is needed, but not to defer the linking of the
module. This is a phase between the "module context attach" phase and the
"evaluation" phase.

As an entirely separate phase, this proposal does not otherwise interact with
the deferred imports proposal.

### Module Expressions & Module Declarations

The module objects defined by the [Module Expressions][] and
[Module Declarations][] proposals, should align with whatever SourceTextModule
phase object foundations are specified in this proposal.

### Compartment Loaders

The [Compartments Proposal][] provides a way to dynamically create module
instances from module source objects, optionally providing custom loaders.

The module source and module instance definitions are being aligned with the
definitions in use within this proposal and others. Where they are specified
in this proposal or others, the compartment loaders proposal may extend their
functionality further in future by adding new methods to these existing objects
for example.

## Q&A

_Post an [issue](https://github.com/lucacasonato/proposal-module-instance-imports/issues)._

[Deferred Imports]: https://github.com/tc39/proposal-defer-import-eval
[Loaders Proposal]: [https://github.com/tc39/proposal-compartments/blob/master/0-module-and-module-source.md]
[Import Attributes Proposal]: https://github.com/tc39/proposal-import-attributes
[Module Expressions]: https://github.com/tc39/proposal-module-expressions
[Module Declarations]: https://github.com/tc39/proposal-module-declarations
[Source Phase Imports]: https://github.com/tc39/proposal-source-phase-imports