# ECMAScript Module Phase Imports

## Status

Champion(s): Luca Casonato, Guy Bedford

Stage: 1

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
function createWorker(url) {
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

A better language primitive for worker loading can improve worker ergonomics
for users as well as their support in analysis and build tooling.

## Proposal

By defining a new phase for ECMAScript module records, it is possible to 
import a handle to a module statically, and pass this handle to the worker:

```js
import source myModule from "./my-module.js";

// `{ type: 'module' }` can be inferred since myModule is a module object
const worker = new Worker(myModule);
```

This technique solves analysis problems (1) and (2) for worker imports in
improving the runtime worker ergonomics - supporting static worker references,
while resolving as module-relative via the normal module resolution rules with
all resolution features supported.

The improved static analysis makes it possible for tools to analyze the worker
references more easily, to determine that a static `myModule` handle is being
passed directly to `new Worker`. Bundling can be performed by replacing the
`./my-module.js` phase import with a phase import to the fully optimized worker
chunk to load.

Defining this phase would then also lay the ground work for the future module
harmony proposals that require a source representation.

Since phases also support a dynamic import form, we also define the dynamic variant:

```js
const workerModule = await import.source('./worker.js');
new Worker(workerModule);
```

## Background

With the recently introduced [Source Phase Imports][], it is now possible to
define import phases that exist prior to the full linking and execution of the
module graph.

These phases form the basis of a number of new proposals for modules including
module declarations, module expressions, deferred module imports, virtualization
through loaders and new worker semantics.

Currently these phases are not yet specified for ECMAScript modules themselves.

One of the driving specification constraints for these objects is how they
behave for workers and other agents, which we would argue forms a critical
design constraint for these features. This is why this proposal's problem
statement is seen as the most suitable "next step" in the larger module harmony
layering efforts, with the phase object or objects specified here to support the
layering of future proposals.

## Design

The current proposed API is for a `ModuleSource` class instance extending `AbstractModuleSource`.

Helper methods are provided to get the direct list of dependencies and exports of the module.

These helper methods are designed to allow for determining the static public exports and public
imports of a module, but do not give information about the internal module identifiers or dynamic
import.

### `ModuleSource.prototype.dependencies()`

Returns the ordered list of the dependencies `Dependency[]`, defined by:

```ts
interface Dependency {
  specifier: string,
  attributes: null | {
    [key: string]: string
  },
  phase: null | 'source'
}
```

This list includes all `import` statements as well as all `export` statements with a `from` clause.

### `ModuleSource.prototype.exports()`

Returns the list of public exports `(Export | StarReexport)[]`, defined by:

```ts
interface Export {
  type: 'export',
  name: string,
}

interface StarReexport {
  type: 'star-reexport',
  from: Dependency
}
```

`Export` is provided for both re-exports and local exports. For example,
`export * as X from './x.js'` would be reflected as the export `{ type: 'export', name: 'x' }`,
and then also appear in the `Dependency` dependency list separately.

### Dynamic Import

On every source, we define a `[[Module]]` internal slot which contains a reference to an
`Source Text Module Record` for the source. This represents an unlinked, uninstantiated module
instance that is uniquely identified for each module source in a given module environment. This
allows ensuring spec identity between sources and instances in the specification, but
implementations may choose to optimize out its creation to be lazy or handled in any other way.

When passing any concrete `AbstractModuleSource` instance into a dynamic `import()` expression,
the module record at its `[[Module]]` internal slot is imported.

### Worker Invocation

The expectation for the HTML integration is that `new Worker(module)` or any concrete instance of
`AbstractModuleSource` would behave as if the module was first transferred into the worker and then
imported with dynamic `import()`.

An additional expectation here is also that the created worker inherits the full resolution rules of
the parent environment that the module source was created from to ensure that module resolution
behaviour remains the same as in the parent context.

## Integration with Other Specifications

### CSP Integration

For WebAssembly, CSP integration is a compile-time check, which occurs before construction of
the `AbstractModuleSource` corresponding to the WebAssembly module. That is, by the time one has
an `AbstractModuleSource` one has already passed the CSP policy checks.

For JavaScript, CSP integration similarly occurs statically before execution, where having `source`
phase import handle to a JS `ModuleSource` implies the CSP permission to execute that source.

For dynamic import, passing an `AbstractModuleSource` to dynamic import would not need to go through
CSP checks, since the obtained object would have already been vetted by CSP.

For `new Worker(module)`, there may be a stricter `worker-src` policy than the `script-src` policy,
requiring CSP policy verification against the `src` for the module. In this case, it should be
possible to recreate the original CSP `src` from the `[[HostDefined]]` data, without needing any
explicit ECMA-262 integration.

### Structured Clone

A `ModuleSource` instance can be supportable in structured clone, since the underlying
source data has no realm-dependence. Any data stored in `[[HostDefined]]` would need to be
defined to itself be structured cloneable to be able to be recreated.

## Layering

### Source Phase Imports

This proposal is designed to work in conjunction with [Source Phase Imports][],
whether or not it directly specifies a source phase for ECMAScript modules.

### Import Attributes

The [Import Attributes Proposal][] provides a way to pass attributes to the
module loader. These attributes are used during source loading and resolution.
Because the module source has already gone through this process, they are
already _attributes-influenced_ by the time their handle is obtained.

When passing a module module source object to a dynamic `import()` or `new Worker`,
any additional `with` attributes would therefore be unsupported - and setting attributes
would throw an error.

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

The module source definition here is being aligned with the definitions in use
within this proposal and others. Where they are specified in this proposal or others,
the compartment loaders proposal may extend their functionality further in future by
adding new methods to these existing objects for example.

## Q&A

_Post an [issue](https://github.com/tc39/proposal-esm-phase-imports/issues)._

[Compartments Proposal]: https://github.com/tc39/proposal-compartments
[Deferred Imports]: https://github.com/tc39/proposal-defer-import-eval
[Loaders Proposal]: [https://github.com/tc39/proposal-compartments/blob/master/0-module-and-module-source.md]
[Import Attributes Proposal]: https://github.com/tc39/proposal-import-attributes
[Module Expressions]: https://github.com/tc39/proposal-module-expressions
[Module Declarations]: https://github.com/tc39/proposal-module-declarations
[Source Phase Imports]: https://github.com/tc39/proposal-source-phase-imports