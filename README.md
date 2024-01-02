# Static Module Source & Instance

## Status

Champion(s): Luca Casonato, Guy Bedford

Stage: 0

## Motivation

### Static Module Analysis

Multiple JavaScript modules together form a module graph, which encodes the
execution behaviour of an application. At runtime, this graph is traversed to
determine which modules are required to be loaded, and in what order.

Outside of the runtime, other tools, such as bundlers, dependency managers, and
static analysis tools like type checkers, operate on this same execution graph.
Because these tools do not execute the code, they need to build the graph
entirely based of static analysis of the code.

In many cases, static analysis of ESM works well, because the majority of
relations between modules are expressed through _static imports_, which use
statically analyzable string literals to refer to other modules. However, there
are various classes of other module relations that are not fully statically
analyzable.

One example of this is dynamic imports, since the module specifier passed into
them may be computed at runtime. In practice, however, tools can often infer
many cases of dynamic import - for example a direct string literal can always be
analyzed:

```js
// Trivial static analysis
import("./my_module.js");
```

More advanced analysis tools can enlist partial evaluation to determine
conditional loading edges and their predicates / substituions, although this is
usually out of scope for most tooling:

```js
// Determinable partial evaluation examples
const moduleSpecifier = "./my_module.js";
import(moduleSpecifier);

const aOrB = unknownCondition ? "a": "b";
import(`./${aOrB}.js`);

// Indeterminate example
import(unknownOutput());
```

The ability to extend analysis to quite complex partial evaluationc cases is
made possible by the fact that we can always locate the _sink_ call of the
dynamic import expression, since `import()` is a syntactical construct and not
an arbitrary function binding.

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
which modules are loaded in workers, and this is a major problem for bundlers
especially, because they need to know which modules are being imported, so they
can bundle them into the output, or emit new entrypoints for entrypoint modules
(as is the case with workers).

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

This proposal builds on top of the phases primitives introduces by the [Source
Phase Imports][] proposal, introducing two new primitives for importing a handle
to a ECMAScript module - the `source` phase and the `module` phase.

### Module Instance

Module instances return a singular stateful object from the module registry that
captures the module source, resolution and evaluation context for an import, but
which is initially unlinked and unevaluated (unless that module had already been
imported previously, in which case it may be in a futher loading state).

Instances import modules at the _module context attach_ phase of the module
loading process, provides a new phase before the _evaluation_ phase (the normal
static import phase), effectively exposed from the internal record already used
to track module loading.

A module instance can at anytime be driven to be fully linked and evaluated in
the current realm by passing them to dynamic `import()`:

```js
// enables a form of static preloading for dynamic import
import module heavyModule from "./heavy-module.js";

function heavyModule() {
  await import(heavyModule);
}
```

Extending `new Worker` to support taking a module instance, we can support
static module workers:

```js
import module workerMod from "./my-worker.js";
// { type: "module" } is inferred since module instances are always modules
const worker = new Worker(workerMod);
```

This solves analysis problems (1) and (2) for worker imports in supporting
static worker references, that resolve through the normal module resolution
rules in being module-relative and supporting all resolution features.

The object representing the module instance is an opaque object called the
`Module` object. It is a constructor-less object that can only be obtained
through the `import module` / `import.module` syntax. It only has one property:

- `get Module.prototype.source()` - Obtain the module source code as a
  `ModuleSource` object.

The `Module` object is not supported by structured clone - instead it can only
be used to create a top-level module worker. This module worker has its baseURL
set to the URL of the instance and also shares its resolution context with the
parent, for example sharing its host-specific resolution state such as the
import map.

### Module Source

Module sources are stateless representations of the source of a module. They
also contain an implicit reference to the original URL of the module, as
host-specific metadata, to support security and capability-based checking.

_Module sources_ then support transfer into workers, for supporting loading
modules statically into existing workers:

```js
import source depSource from 'dep';
worker.postMessage(depSource);
```

For now, module sources only support canonical instancing. That is, in a given
environment, every module source has a _canonical instance_, since all module
sources are created statically they always have a corresponding instance in this
way.

When loading the _canonical instance_ of a module source in a different context,
it is keyed by the double key of both the original key of the module source's
canonical instance in the original context, and also the key of the context
reference itself. This way all sources have a unique representation in every
context, even if they happen to share the same URL.

When passed to dynamic `import()` we obtain the canonical instance for a given
module source:

app.js
```js
// create a new worker, then pass a module into it
import workerMod from './worker.js';
import source depSource from 'dep';
const worker = new Worker(workerMod);
worker.postMessage(depSource);
```

worker.js
```js
addEventListener('message', evt => {
  const depSource = evt.data;
  // dynamically import a new instance from the passed source
  const dep = import(depSource);
});
```

The `ModuleSource` prototype has the following helper methods to allow for
access to the static analysis on module graphs:

* `get ModuleSource.prototype.staticImports()`, returning a list of objects of
the form `StaticImport[]`, where `StaticImport` is defined as the object `{
specifier: string, phase: string, attributes?: [string, primitive][] }`, the
analysis of an import.
* `get ModuleSource.prototype.staticExports()`, returning a list of objects of
the type `DirectExport | ReExport | StarReExport`, with the type of
`DirectExport`, `{ type: 'direct', name: string }`, the type of `ReExport`, `{
type: 're-export', name: string, imported: string, import: StaticImport }`, and
the type of `StarReExport`, `{ type: 'star-re-export', import: StaticImport }`.

In this way it can also be used as an analysis-building helper in other tools
for simple analysis cases.

### Dynamic Phase Imports

Since phases also support a dynamic import form, we also get the dynamic
variants of these phases:

```
const workerModule = await import.module('./worker.js');
const depSource = await import.source('./dep.js');
```

with these dynamic forms solving analysis problem (3) as stated in the
motivation.

## Interactions

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

This proposal is designed to work in conjunction with "source" phase imports.
Imports at the source phase are designed to be used for importing stateless
module source objects, that can then be multiply instantiated into stateful
module instances. While neither the source phase imports proposal, nor this
proposal propose the multiple instantiation of module instances, this is a
direction we are exploring in the loaders proposal. You can find more concrete
examples in the loaders section below.

This proposal does allow for synchronous access to a source by using the
`source` property on the module instance object.

It is also possible to pass a module instance to `import.source` instead of a
module specifier.

### Module expressions & declarations

The module instance object returned from module instance imports is the same
module instance object created by module expressions. Both proposals provide
means to get a module instance object, but they do so in different ways:

- Module expressions and create a module instance object from source text in the
  current module.
- Module instance imports create a module instance object from source text in a
  different module, referenced by a module specifier.

The ways this object is used is identical between the two proposals. In both
cases, the module instance object is passed to APIs that accept module
specifiers currently, such as `import()` or `new Worker()`.

The module declaration proposal also provides a parallel binding namespace for
statically determinable module instance objects to be used in `import from`
static syntax. Module instances imported statically via module instance imports
would also be present in this namespace, so could also be imported from
statically using the `import from` syntax.

Since module expressions and module declarations are statically rooted in the global
registry (not GC'd), they would also be able to be passed to `new Worker()`.

### Loaders

The loaders proposal provides a way to dynamically create module instances from
module source objects, optionally providing a custom loader and `import.meta`.
The module instances created by loaders are the same `Module` object created by
module instance imports.

These types of instances would be unrooted to the static graph and therefore GC'd
and existing as dynamically created instances that might bind to arbitrary imports.
As a result since these instances are unrooted in the global module registry, and
thus have no concept of identity or keying across contexts, they would not be
transferrable into workers and throw an error in this case.

The loaders proposal extends the module instance object provided by this
proposal with a constructor that takes a module source. This allows for dynamic
creation of module instances.

This constructor unlocks multiple instantiation of a module instance, by calling
the constructor multiple times with the same module source. This is useful to
create isolated module instances, for example to use in testing.

The module instances created by loaders are not linked or evaluated immediately,
just like module instance imports. They can be linked and evaluated later by
passing them to `import()`, just like module instance imports.

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

This is where module instances can be useful. The main thread (trusted) can load
a module from disk or network, from audited/trusted sources, and then pass the
module instance to the worker thread to be executed. The worker thread can then
evaluate the module instance, without having to load any modules from disk or
network, or evaluate any arbitrary strings. In essence, the worker thread can
only execute code that was audited by the main thread. It get's access to this
evaluation capability by being passed a module instance, so the module instance
is a capability object.

Together with the [loaders
proposal](https://github.com/tc39/proposal-compartments) this proposal can
provide novel sandboxing techniques using workers.

## Q&A

_Post an [issue](https://github.com/lucacasonato/proposal-module-instance-imports/issues)_

[Source Phase Imports]: https://github.com/tc39/proposal-source-phase-imports