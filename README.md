# ECMAScript Module Phase Imports

## Status

Champion(s): Luca Casonato, Guy Bedford

Stage 2.7

The proposal spec text is currently based to the ECMA-262 PR for Source Phase Imports at https://github.com/tc39/ecma262/pull/3492.

## Problem Statement

This proposal seeks to solve the _static worker_ module analysis problem for
JavaScript, through defining a source phase import for Source Text Module.

## Background

With the recently introduced [Source Phase Imports][], it is now possible to
define import phases that exist prior to the full linking and execution of the
module graph.

While WebAssembly is supported, the exact semantics of the source phase for
ECMAScript modules themselves is not yet specified.

One of the driving specification constraints for these objects is how they
behave for workers and other agents, which we would argue forms a critical
design constraint for these features. This is why this proposal's problem
statement is seen as the most suitable "next step" in the larger module harmony
layering efforts, with the phase object or objects specified here to support the
layering of future proposals, including module expressions, module declarations
and virtualization through compartments loaders.

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

By defining the source phase for ECMAScript module records, it is possible to 
import a handle to a module statically, and use it to directly initialize the worker:

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

Defining the source phase then also lays the ground work for the future module
harmony proposals that require a source representation.

Since phases also support a dynamic import form, we also define the dynamic variant:

```js
const workerModule = await import.source('./worker.js');
new Worker(workerModule);
```

The current proposed API is for a `ModuleSource` class instance extending `AbstractModuleSource`.

This is a new non-global intrinsic with the same reachability properties of
`AbstractModuleSource` via any source phase import to a JS module.

## Semantics

The proposal defines the following new semantics:

1. The new `ModuleSource` intrinsic, exposed on the global object.
2. Source phase imports for `Source Text Module Record` - `import source mod from './mod.js'`.
3. Dynamic import of a module source - `import(mod)` both where `mod` is a `ModuleSource` instance **and** `mod` is a `WebAssembly.Module` instance.
4. The serialization and transfer semantics of `ModuleSource` between agents, via both `postMessage` and `structuredClone`.

Ensuring well-defined module keying semantics is thus a critical aspect of this proposal.
In addition, transfer semantics and security integration semantics also relate strongly to the keying
model.

### The ModuleSource Object

A `ModuleSource` instance contains an internal `[[SourceTextModuleRecord]]` slot which points to its underlying module record.

The `ModuleSource` constructor is not constructible and throws a TypeError error if attempting to be constructed. There
are currently no plans to make the `ModuleSource` constructor available, since this is deemed to be a new evaluation vector.

Instead, the ways to obtain a `ModuleSource` instance consisting of a valid `[[SourceTextModuleRecord]]` slot is via:

1. `import source myModule from '/module.js'` source phase imports
2. `module myModule { ... }` module declarations in future, when that proposal adds this (which implies `eval('module myModule { ... } myModule'`)
3. Via serialization in worker messages or via `structuredClone(myModule)`.

### Dynamic Import

Since `ModuleSource` points to its `[[SourceTextModuleRecord]]`, `import(moduleSource)` of a module source is defined as
importing that module record.

### Structured Clone

When applying `structuredClone(module)`, not only is a new `ModuleSource` instance created, but a new `[[SourceTextModuleRecord]]` is also created
due to the serialization and deserialization of the module record itself.

This way, structured clone results in multi-instancing of the module:

```js
import source module from '/module.js';
let module2 = structuredClone(module);
await import(module) !== await import(module2);
```

While not part of this specification, the unique identity of each module source, despite having the same URL key is what will also
allow multiple module declarations to exist in the same module with the same URL while having separate instances:

```js
// SYNTAX NOT SPECIFIED HERE - See Module Declarations Proposal
module moduleA {
  export let url = import.meta.url;
}
module moduleB {
  export let url = import.meta.url;
}

const [moduleA, moduleB] = await Promise.all(import(moduleA), import(moduleB));
moduleA === moduleB // false
moduleA.url === import.meta.url // true
moduleA.url === moduleB.url // true
``` 

### Agent Transfer

Agent transfer needs to support populating registry source text with transferred sources.

For example consider transferring a source and importing it from a worker:

```js
import source mod from '/mod.js';

const worker = new Worker('/worker.js');
worker.postMessage(mod);
```

Where `/worker.js` contains:

```js
onmessage = ({ mod: data }) => {
  const instance1 = await import(mod);
  // Later on:
  const instance2 = await import('/mod.js');
  instance1 === instance2 // true

  const source = await import.source('/mod.js');
  source === mod // true
};
```

The worker does not need to fetch the source for `mod.js` since it was serialized and transferred with
the `ModuleSource` record.

Later, when we import the string specifier `/mod.js` directly, we obtain an instance that is the same as the
one that was imported, since the `import(mod)` itself injects the module key for `mod` into the registry.

We can see this by importing the source phase import for `/mod.js` - that it returns the source we provided.

But what if the worker had imported `/mod.js` first? And what if the worker's network request was made later on
and it received a different source text for `/mod.js` over the wire?

```js
onmessage = ({ mod: data }) => {
  const instanceA = await import('/mod.js');
  // Later on:
  const instanceB = await import(mod);
  instanceA === instanceB // false

  const source = await import.source('/mod.js');
  source === mod // false
};
```

What happens here instead is that the "canonical instance" in the module registry is defined by the first
importer, so that when the module source version received via `postMessage` is imported, it is not
able to return the same instance.

We define a concrete method in the spec, `ModuleSourcesEqual` which is able to perform source text
equality to perform coalescing here _only if_ the exact source text is the same.

### Module Keying

In the current ECMA-262 specification, the `HostLoadImportedModule` idempotency property defines _module keying semantics_.
The guarantee here is that if two imports are made to the same `Module Request` (specifier and attributes list),
that the same `Module Record` must be returned.

When supporting `ModuleSource` in dynamic import, we effectively extend this property to Module Source imports, by defining
the concept of _module source equality_.

The rule here then is based on registry coalescing, and supporting multiple modules with the same URL in the registry:

1. When a module source is imported, its URL is injected into the module registry, unless another module already exists
  at that URL.
2. If a module already exists at the URL, we check if the module source for that module exactly matches the source text
  of the module source of the module source being imported, and if it does we return the existing instance.
3. If the existing instance does not exist, the module is not added to the registry at all, instead it is uniquely instanced
  based on the identity of the module source itself. In this sense there is a secondary registry consisting of a weak map from
  module source objects to module instances.

## Web Spec Integrations

### Web Workers

The expectation for the HTML integration is that `new Worker(module)` or any concrete instance of
`AbstractModuleSource` would behave as if the module was first cloned into the worker and then
imported with dynamic `import()`.

An additional goal here would be for workers constructed this way to automatically inherit the same
resolution rules of the parent environment that the module source was created from to provide
consistent module resolution.

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

## Module Harmony Layering

This spec is designed to layer with future module harmony proposals, specifically
with [Module Declarations][] and [Compartments][].

### Module Declarations

Module declarations permit the definition of nested modules, whose runtime
representation is designed here to be `ModuleSource`:

```js
module myModule {
  import 'dependency';
  export function main () {
  }
}

myModule instanceof ModuleSource; // true
import(myModule);
```

The module keying model for `ModuleSource` thus needs to take into account
that multiple modules may share a URL.

### Compartments

The [Compartments Proposal][] provides a way to dynamically create module
instances from module source objects, optionally providing custom loaders.

The module source definition here is being aligned with the definitions in use
within this proposal and others. Where they are specified in this proposal or others,
the compartment loaders proposal may extend their functionality further in future by
adding new methods to these existing objects for example.

The module keying semantics designed in this specification would extend to multi
registry keying when used with compartments.

## Q&A

### What were the Stage 2 and Stage 2.7 considerations for this proposal?

See the former status updates at https://github.com/tc39/proposal-esm-phase-imports/issues/50.

### What was source analysis removed from this proposal?

This proposal originally also included adding source analysis properties to the module source
of the form:

* `AbstractModuleSource.prototype.imports: () => Import[]`
* `AbstractModuleSource.prototype.exports: () => Export[]`
* `AbstractModuleSource.prototype.hasImportMeta: boolean`
* `AbstractModuleSource.prototype.hasTopLevelAwait: boolean`

Where `Import` was defined by:

```ts
interface Import {
  specifier: string,
  phase: null | 'source'
}
```

and `Export` was defined by `DirectExport | Reexport | ReexportAll`:

```ts
interface DirectExport {
  type: 'direct',
  names: string[]
}

interface Reexport {
  type: 'reexport',
  name: string,
  import: string | null, // null used to indicate a namespace reexport
  from: string,
}

interface ReexportAll {
  type: 'reexport-all',
  from: string,
}
```

This feature was removed due to it causing this proposal to have dual motivations.

JS module source analysis may be remotivated in future on JS virtualization work such as compartments.

That this feature was removed in this initial proposal does not by any means indicate that this
source analysis is not viable in future, only that a narrower focus on use cases was needed for this
proposal.

_Post an [issue](https://github.com/tc39/proposal-esm-phase-imports/issues)._

[Compartments Proposal]: https://github.com/tc39/proposal-compartments
[Deferred Imports]: https://github.com/tc39/proposal-defer-import-eval
[Loaders Proposal]: [https://github.com/tc39/proposal-compartments/blob/master/0-module-and-module-source.md]
[Import Attributes Proposal]: https://github.com/tc39/proposal-import-attributes
[Module Expressions]: https://github.com/tc39/proposal-module-expressions
[Module Declarations]: https://github.com/tc39/proposal-module-declarations
[Source Phase Imports]: https://github.com/tc39/proposal-source-phase-imports
