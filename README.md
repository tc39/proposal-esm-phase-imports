# Module Instance Imports

## Status

Champion(s): Luca Casonato

Stage: 0

## Motivation

Multiple JavaScript modules together form a module graph. This module graph is a
graph of how all modules are related to each other. At runtime, this graph is
built to determine which modules are required to be loaded, and in what order.

Outside of runtime, other tools, such as bundlers, dependency managers, and
other static analysis tools such as type checkers, also need to build this
graph. Because these tools do not execute the code, they need to build the graph
entirely based of static analysis of the code.

In many cases, static analysis of ESM works well, because the majority of
relations between modules are expressed through "static imports", which use
statically analyzable string literals to refer to other modules. However, other
ways that modules can be related to each other are not well statically
analyzable.

Dynamic imports are one example of this. Dynamic imports are in principle not
analyzable without execution, because the module specifier passed into them may
be computed at runtime. However, in practice, many tools can statically analyze
many dynamic imports, because the module specifier is often a string literal
expression placed directly into the `import()` call. Sometimes however, the
module specifier is static (a non computed string literal), but assigned to a
binding before being passed into `import()`. In this case some tools can analyze
the dynamic import, but others can not.

```js
// Impossible static analysis - runtime computed module specifier
const moduleSpecifier = `./my_module_${Math.random()}.js`;
import(moduleSpecifier);

// Trivial static analysis
import("./my_module.js");

// Difficult static analysis - only some tools support this
const moduleSpecifier = "./my_module.js";
import(moduleSpecifier);
```

These cases are all still approachable for static analysis, because `import()`
is not a function call, but a syntactic construct. This means that you can not
do `const import = myImportFunction;` or `const myImport = import;` to alias the
`import()` function.

`new Worker()` is a case where the relation between two modules is so dynamic
that static analysis becomes very difficult and often un-reliable. `new Worker`
is an instance creation where the module specifier is passed in as an argument:

- The module specifier can be passed in both as a string literal, or via a
  binding like shown above for dynamic imports.
- `Worker` is a global variable, so a different binding can be set to the value
  of `Worker`. For example, `const Worker2 = Worker;`.
- `Worker` is a constructor, so it can be subclassed, and the subclass can be
  used to create a worker. For example, `class MyWorker extends Worker {}`.

Additionally, because because `Worker` does not use dynamic scoping, you can not
directly pass a relative module specifier into a worker. Instead, to load a
module relative to the current module, you need to use `import.meta.url` /
`import.meta.resolve` to resolve the module specifier to an absolute URL before
passing it into the worker.

This results in a situation where it is very difficult to statically analyze
which modules are related to each other via `new Worker()`. This is a problem
for bundlers especially, because they need to know which modules are being
imported, so they can bundle them into the output, or emit new entrypoints for
entrypoint modules (as is the case with workers).

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

The language is lacking a primitive to deal with this.

## Use cases

The lack of more static analysis friendly primitives for determining relation
between modules is a problem for various forms of bundlers, dependency managers,
and other static analysis tools. Here are some "in-the-wild" examples of this
problem:

### Worker bundling in `esbuild`

`esbuild` does not have smart enough analysis to pick up on
`new Worker(new URL("./my_worker.js", import.meta.url), ...)` style worker
usage, so it is not able to automatically bundle workers. Instead, it requires
you to manually specify the worker as a separate entrypoint in your project, and
then manually configure your `new Worker` call to load this entrypoint.

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

Together with the
[loaders proposal](https://github.com/tc39/proposal-compartments) this proposal
can provide novel sandboxing techniques using workers.

## Proposed solution

This proposal introduces a new import syntax, which allows importing modules at
the "module context attach" phase of the module loading process, instead of at
"evaluation" phase (which is the phase that static imports are imported at), or
["source" phase](https://github.com/tc39/proposal-source-phase-imports).

Imports at this phase return a module instance. A module instance is an
structure that combines a module source with resolution and evaluation context.
A module instance is initially unlinked and unevaluated. A module instance can
later be linked and evaluated. Current "evaluation" phase imports also make use
of this module instance internally, with the difference that modules imported to
"evaluation" phase are always linked and evaluated (as opposed to this proposal
where they are not yet linked and evaluated).

In JavaScript, module instances are represented as objects. These objects can be
passed to APIs that accept module specifiers, such as `import()` or
`new Worker()`. `import()` directly links and evaluates the module instance
passed to it, returning the module namespace. `new Worker()` would structurally
clone the module instance, and then link and evaluate the cloned module instance
in the worker.

```js
import module heavyModule from "./heavy_module.js";

function heavyModule() {
  await import(heavyModule);
}
```

```js
import module workerMod from "./my_worker.js";

const worker = new Worker(workerMod, { type: "module" });
```

There is also a dynamic import form of this syntax:

```js
const workerMod = await import.module("./my_worker.js");

const worker = new Worker(workerMod, { type: "module" });
```

The object representing the module instance is an opaque object called the
`Module` object. It is a constructor-less object that can only be obtained
through the `import module` / `import.module` syntax. It only has one property:

- `module.source` - The module source code as a `%AbtractModuleSource%` object.

The `Module` object is structured cloneable and serializable, meaning it can be
passed to other contexts, such as Web Workers, and then linked and evaluated in
that context. Identity does not round-trip through structured clone to support
GC-ability of module instances. Linkage and evaluation of a module instance is
not preserved when cloning. The resolver does not transfer with the module
instance - the resolver of the context the module instance is cloned into is
always used.

## Interactions

### Import attributes

The import attributes proposal provides a way to pass attributes to the module
loader. Just like with "source" phase imports, this proposal composes with the
import attributes proposal, allowing attributes to be passed on module instance
imports.

The attributes are used during source loading and resolution, like specified for
"source" phase imports. The module instance has already gone through this
process, so it is already "attributes-influenced". This means that when you pass
a module instance to a dynamic import or `new Worker`, you can not pass
additional `with` attributes.

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

### Loaders

The loaders proposal provides a way to dynamically create module instances from
module source objects, optionally providing a custom loader and `import.meta`.
The module instances created by loaders are the same `Module` object created by
module instance imports.

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
through the addition of support for module instances in the
`await import.defer()` syntax.

## Q&A

_Nothing yet!_
