# Phase Imports for SourceTextModule

## Status

Champion(s): Luca Casonato, Guy Bedford

Stage: 0

## Background

With the recently introduced [source phase]() imports, it is now possible to
define import phases that exist prior to the full link and execution of the
module graph.

These phases form the basis of a number of new proposals for modules including
module declarations, module expressions, deferred module imports, virtualization
through loaders and new worker semantics.

Currently this phase is not yet specified for ECMAScript modules themselves,
as we do not have a definition of this object yet for JavaScript.

One of the driving specification constraints for these objects is also how they
behave when transferred between agents and workers, which we would argue forms
a critical design space for these features.

## Problem Statement

This proposal seeks to solve the "static worker" module problem for JavaScript,
through defining a suitable source phase import for SourceTextModule.

### The Worker Static Module Analysis Problem

`new Worker()` is a case where the relation between two modules suffers from a
number of analysis issues:

1. It always takes an arbitrary expression to locate the worker URL, like
   dynamic `import()` does. We thus never have fully static worker includes like
   we do for normal ESM imports and workers therefore always fall under some
   kind of advanced expression analysis / partial evaluation case in tooling.
2. The string passed to `new Worker(url)` doesn't support module resolution
   rules - relative URLs are not resolved module-relative but baseURL-relative,
   which will be different at runtime versus build / analysis time. For this
   reason it is often impossible to locate workers without an out-of-band
   baseURL-based rooting configuration for the analysis system and for the
   runtime system. The best pattern here is to first resolve URLs using
   `import.meta.resolve()` or `import.meta.url`, but there are many different
   patterns here and no single standard approache employed by developers.
3. `Worker` is a global variable, unlike dynamic `import()`, so that we can't
   even rely on that binding as a reliable sink for worker URL invocation
   expression analysis.

This results in a situation where it is very difficult to statically analyze
which modules are loaded in workers. This causes two major problems:

1. Runtimes are unable to preload or optimize the worker loading until it is
dynamically executed, resulting in the waterfall for a module worker happening
very late.
2. Bundlers and tools are unable to know statically which modules are being
imported, so they can bundle them into the output, or emit new entrypoints
for entrypoint modules (as is the case with workers).

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

A better language primitive for worker loading can resolve all three of these
static analysis gaps improving the situation for analysis, build and runtime
tooling.

## Proposal

By defining a new phase for ECMAScript module records, it is possible to 
import a handle to a module statically, and pass this handle to the worker:

```js
// enables a form of static preloading for dynamic import
import somephase myModule from "./my-module.js";

// `{ type: 'module' }` can be inferred since myModule is a module object
const worker = new Worker(myModule);
```

This technique would solve analysis problems (1) and (2) for worker imports in
supporting static worker references, that resolve through the normal module
resolution rules in being module-relative and supporting all resolution features.

Runtimes and tooling would be able to statically see what module loading waterfall
would be happening and provide ahead of time analysis - at runtime to enable
performance improvements, and for tools to provide stronger static guarantees.

In addition this new phase would then also layer with the future proposals for
inline modules and virtualization.

### Module Instance v Module Source

There is an outstanding design question as to whether this phase should be an
instance or source phase for the module.

The instance phase represents an entire graph of modules, while the source phase
represents just a specific module source without its linking being defined.

### Transferability

The phase defined for ECMAScript modules may or may not be a transferable phase,
via `postMessage` into the worker. While source phase is currently transferable,
instance phase may have more difficulties in transferability due to having to
handle edge cases around dependency transfer.

### Dynamic Phase Imports

Since phases also support a dynamic import form, we also get the dynamic
variant:

```
const workerModule = await import.somephase('./worker.js');
new Worker(workerModule);
```

with these dynamic forms solving analysis problem (3) as stated in the
motivation.

## Layering

### Import attributes

The import attributes proposal provides a way to pass attributes to the module
loader. Just like with "source" phase imports, this proposal composes with the
import attributes proposal, allowing attributes to be passed on module instance
imports.

The attributes are used during source loading and resolution, like specified for
"source" phase imports. The module source and module instance has already gone
through this process, so it is already "attributes-influenced". This means that
when you pass a module instance or module source to a dynamic `import()` or `new
Worker`, you can not pass additional `with` attributes, and setting attributes
in `import()` would throw an error.

### Source phase imports

This proposal is designed to work in conjunction with "source" phase imports,
whether or not it directly specifies a source phase for ECMAScript modules.

### Module expressions & declarations

The module object returned from the phased import should be based on the same
foundations as the module objects of module expressions and module declarations.

### Loaders

The loaders proposal provides a way to dynamically create module instances from
module source objects, optionally providing a custom loader and `import.meta`.
The module instances created by loaders are the same `Module` object created by
module instance imports.

The loaders proposal should be layered to use the new phase defined here in
virtualization, when enabling the creation of dynamic module instances.

### Deferred imports

The deferred imports proposal provides a way to defer the synchronous evaluation
of a module until it is needed, but not to defer the linking of the module. This
is a phase between the "module context attach" phase and the "evaluation" phase.

This differs from this proposal, because this proposal does not perform eager
linkage. This is important to support passing module instances to other contexts
such as Web Workers without the linkage being performed in the current context.

This proposal does not interact with the deferred imports proposal, except
through the addition of support for module instances in the `await
import.defer()` syntax.

## Use Cases

Supporting new static analysis-friendly primitives for managing more complex
module loading scenarios extends static analysis tools to new uses, including
the following "in the wild" examples:

### Worker bundling in `esbuild`

`esbuild` does not have smart enough analysis to pick up on `new Worker(new
URL("./my_worker.js", import.meta.url), ...)` style worker usage, so it is not
able to automatically bundle workers. Instead, it requires you to manually
specify the worker as a separate entrypoint in your project, and then manually
configure your `new Worker` call to load this entrypoint.

This has various drawbacks:

- Code can not run un-bundled (for example in development), because the code is
  dependant on the bundler emitting a separate entrypoint for the worker at a
  specific location. This means that bundling can not just be a transparent
  performance optimization, but rather requires tool specific codebase changes.
- Significant manual configuration is required on the users part, making the
  barrier to entry higher for using workers. Generally, we should encourage use
  of workers to offload work from the main thread that is used for rendering.

### Dynamic import / worker support in `deno compile`

`deno compile` is a feature of Deno that allows running a JavaScript program as
a standalone executable binary. This works by performing static analysis on the
module graph during compilation, serializing all modules _as is_ into a special
container format [eszip](https://github.com/denoland/eszip), and then embedding
this in a binary.

Upon execution, this binary is then able to load the modules individually from
the container format, and execute them. No code transformation is performed on
the source code - all code is executed as is was found on disk prior to
compilation.

Because at runtime only code embedded in the binary can be loaded, all imported
code must be statically analyzable. If you want to use dynamic imports to defer
a slow executing module until it is needed, or want to use web workers to
offload work from the main thread, you can not do so, because the module was
never loaded into the binary because it was not found during static analysis.

`deno compile` provides a `--include` flag to enable manually specifying
additional modules to be treated as entrypoints, but this is a manual process
that requires you to specify workers and dynamic imports both in code, and in a
build script. Errors from mis-configuration can not be caught at compile time
(because lack of static analysis, duh!), so necessarily result in runtime
exceptions, which is subpar.

### Dependency graph visualization in Deno

Deno provides a `deno info` command that allows you to visualize the dependency
graph of a module, starting at it's entrypoint. This is useful for understanding
the structure of a codebase, and for debugging dependency cycles.

However, because `deno info` only performs static analysis, it can not visualize
the dependency graph of modules that are referenced in ways that are not
trivially statically analyzable. This includes dynamic imports, and workers.

### Dynamic I/O permissions required to start workers in Deno

Deno provides a relatively sophisticated permissions system, that allows you to
lock down which I/O operations can be performed at runtime. It can be configured
through flags passed to the `deno` executable, or dynamically using user
prompts.

When a user performs an import of a local file on disk, this is considered I/O
and requires the relevant file system permission to be set. There is however a
special bypass to this permission check for the entrypoint module, all modules
that are statically reachable from that entrypoint. This means that if you have
a file `./main.ts` that imports a file `./b.ts`, you do not need to have read
permissions for either `./main.ts` or `./b.ts` to execute this entrypoint. This
is safe, because a user can visualize all files that are statically reachable
from the entrypoint before execution (using `deno info`).

There is no static analysis for modules used to start workers, and as such there
is no way to know up front that that given module is part of the module graph,
which means it can not be exempted from the permissions check like static
imports can. This means that starting a worker always requires a permission
check for that file, and all files it imports.

This makes building granular allow-lists of files that are allowed to be read
very difficult.

This also significantly increases the barrier of entry for using workers,
because they don't just work "as is", like regular static imports. This is bad,
because we want to encourage users to run heavy compute tasks off-main-thread.

### Third-party modules using workers

All of the problems described above get exponentially worse when thinking about
workers inside of third party libraries. When a worker is used inside of a third
party library, and a bundler or other static analysis tool can not pick it up
automatically, this requires the user to manually tweak their build tooling to
explicitly specify workers internal to third party dependencies.

This is so cumbersome for end users that library authors often avoid using
workers entirely, even when it would be beneficial to do so, because of the
significant added complexity to end-users. This is bad, because we want to
encourage library authors to use workers to offload heavy compute tasks from the
main thread.

The Wasm build of `esbuild` itself is a good example of a workaround that is
very frustrating for users. It embeds worker source code in a JS file as a
string literal, and then passes this string to a Worker constructor using a blob
URL (see line 1813 of
https://unpkg.com/browse/esbuild-wasm@0.19.2/lib/browser.js). This results in
all stack traces from within this worker to be completely useless, because it
points to the blob URL, and not a readable source file.

### Capability based imports

Capability based imports is a new security primitive that arises from this
proposal. Let's imagine a scenario with two separate JavaScript realms, one on
the main thread, and one on a worker thread. The main thread worker has broad
permissions, and is a trusted hypervisor. The worker thread is untrusted, and
has limited permissions. It does not allow `eval` or other dynamic code
execution, and does not allow access to the file system or network. The question
that arises now is: "How can the worker thread execute any code, if it can't
evaluate strings, and can't load any modules from disk or network?"

This is where module phases can be useful. The main thread (trusted) can load
a module from disk or network, from audited/trusted sources, and then pass the
static module phase to the worker thread to be executed. The worker thread can
then evaluate the module, without having to recheck permissions or evaluate any
arbitrary strings. In essence, the worker thread can only execute code that was
already audited by the main thread. It gets access to this evaluation capability
by being passed a module phase, so the module phase is a capability object.

Together with the [loaders proposal](https://github.com/tc39/proposal-compartments)
this proposal can provide novel sandboxing techniques using workers.

## Q&A

_Post an [issue](https://github.com/lucacasonato/proposal-module-instance-imports/issues)_

[Source Phase Imports]: https://github.com/tc39/proposal-source-phase-imports