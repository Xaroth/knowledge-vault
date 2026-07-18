# Tools

First-party tools that consume the [resource distribution](../index.md) system.
Both resolve logical `res:/` paths to CDN assets by walking the same index chain;
they differ in what they hand back and when.

* [eve-resfile](./eve-resfile.md) - a build-tooling package (Vite / Rollup / PostCSS plugins) that resolves `res:/` imports at build and dev time.
* [eve-resfile-proxy](./eve-resfile-proxy.md) - a local HTTP proxy that serves the whole resource tree as a browsable read-only filesystem.
