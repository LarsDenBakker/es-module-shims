## ES Module Shims

[85% of users](https://caniuse.com/#feat=es6-module) are now running browsers with baseline support for ES modules.

But a lot of the useful features of modules come from new specifications which either aren't implemented yet, or are only available in some browsers.

_It turns out that we can actually polyfill most of the newer modules specifications on top of these baseline implementations in a performant 3.5KB shim._

This includes support for:

* Dynamic `import()` shimming when necessary in eg older Firefox versions.
* `import.meta` and `import.meta.url`.
* [Import Maps](https://github.com/domenic/import-maps) support.
* Importing Web Assembly (note, there is an [open issue on how to handle the 4KB imposed limit](https://github.com/guybedford/es-module-shims/issues/1))

Because we are still using the native module loader the edge cases work out comprehensively, including:

* Live bindings in ES modules
* Dynamic import expressions (`import('src/' + varname')`)
* Circular references, with the execption that live bindings are disabled for the first unexecuted circular parent.

Due to the use of a dedicated JS tokenizer for ES module syntax only, with very simple rewriting rules, transformation is instant.

### Import Maps

> It is also possible to ship import maps directly in Chrome (without using this project!) through the [Chrome Origin Trial](https://developers.chrome.com/origintrials/#/view_trial/3175037737296199681). The hope is for this project to eventually become a true polyfill for import maps in older browsers, but this will only happen once the spec is implemented in more than one browser and demonstrated to be stable.

In order to import bare package specifiers like `import "lodash"` we need [import maps](https://github.com/domenic/import-maps), which are still an experimental specification.

Using this polyfill we can write:

```html
<!doctype html>
<!-- either user "defer" or load this polyfill after the scripts below-->
<script defer src="es-module-shims.js"></script>
<script type="importmap-shim">
{
  "imports": {
    "test": "/test.js"
  },
  "scopes": {
    "/": {
      "test-dep": "/test-dep.js"
    }
  }
}
</script>
<script type="module-shim">
  import test from "test";
  console.log(test);
</script>
```

All modules are still loaded with the native browser module loader, just as Blob URLs, meaning there is minimal overhead to using a polyfill approach like this.

### Dynamic Import

Dynamic `import(...)` within any modules loaded will be rewritten as `importShim(...)` automatically
providing full support for all es-module-shims features through dynamic import.

To load code dynamically (say from the browser console), `importShim` can be called similarly:

```js
importShim('/path/to/module.js').then(x => console.log(x));
```

### Web Assembly

To load Web Assembly, just import it:

```js
import { fn } from './test.wasm';
```

Web Assembly imports are in turn supported.

Import map support is provided both for mapping into Web Assembly URLs, as well as mapping import specifiers to JS or WebAssembly from within WASM.

### Module Workers

To load workers with full import shims support, the `WorkerShim` constructor can be used:

```js
const worker = new WorkerShim('./module.js', {
  type: 'module',
  // optional import map for worker:
  importMap: {...}
});
```

This matches the specification for ES module workers, supporting all features of import shims within the workers.

> Module workers are only supported in browsers that provide dynamic import in worker environments, which is only Chrome currently.

## Implementation Details

### Tokenizer Rewriting

* Tokenizing handles the full language grammar including nested template strings, comments, regexes and division operator ambiguity based on backtracking.
* Rewriting is based on fetching the sources, turning them into BlobURLs and executing up the graph.
* When executing a circular reference A -> B -> A, a shell module technique is used to "shim" the circular reference into an acyclic graph. As a result, live bindings for the circular parent A are not supported, and instead the bindings are captured immediately after the execution of A.
* The approach will only work in browsers supporting ES modules.
* CSP is not supported as we're using fetch and modular evaluation.

### Import Maps
* The import maps specification is under active development and will change, all of the current specification features are implemented, but the edge cases are not currently fully handled. These will be refined as the specification and reference implementation continue to develop.

### Web Assembly
* In order for Web Assembly to execute in the module graph as a blob: URL we need to use `new WebAssembly.Instance` for synchronous execution, but this has a 4KB size limit in Chrome and Firefox which will throw for larger binaries. There is no known workaround currently. Tracking in https://github.com/guybedford/es-module-shims/issues/1.
* Exports are snapshotted on execution. Unexecuted circular dependencies will be snapshotted as empty imports. This matches the [current integration plans for Web Assembly](https://github.com/WebAssembly/esm-integration/).

## Inspiration

Huge thanks to Rich Harris for inspiring this approach with [Shimport](https://github.com/rich-harris/shimport).

### License

MIT
