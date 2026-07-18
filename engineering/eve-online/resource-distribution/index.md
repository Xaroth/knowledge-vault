# Resource distribution

How the EVE Online client's assets — everything from UI icons and textures to the
executable's own distribution files — are published to CDNs and resolved by name.
The client never ships its assets in-place; it downloads them from two CDN hosts,
guided by a chain of plain-text index files that map human-readable logical paths
(`res:/ui/texture/icons/...`) to content-addressed, hash-named blobs.

These pages describe that system as observed from the outside, and the two
first-party tools that consume it.

# Reading order

* [Overview](./overview.md) - the two CDN hosts, the binaries-vs-resources split, and the end-to-end path-resolution chain.
* [Build discovery](./build-discovery.md) - how a client build number is found from a server name, servers (TQ / SISI), platforms, and protected builds.
* [Index file format](./index-format.md) - the two index tiers, the `app:/resfileindex.txt` indirection, and the exact comma-delimited line format.
* [CDN cache paths](./cdn-cache.md) - content-addressed hash filenames, MD5 checksums, sizes, and the launcher-compatible on-disk cache layout.

# Tools

* [Tools](./tools/index.md) - the first-party consumers of this system: the `eve-resfile` bundler package and the `eve-resfile-proxy` HTTP server.

# Source material

This area is derived from the source of two first-party tools, read as the
authoritative model of the system:

* `@eve-online-tools/eve-resfile` — `packages/eve-resfile` in
  [github.com/eve-online-tools/node-packages](https://github.com/eve-online-tools/node-packages).
* `eve-resfile-proxy` —
  [github.com/eve-online-tools/eve-resfile-proxy](https://github.com/eve-online-tools/eve-resfile-proxy).
