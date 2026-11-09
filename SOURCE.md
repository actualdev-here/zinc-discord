# Source and compatibility

Version: 0.1.0 (experimental).

Exported from [Zinc commit 6baeeb0025b264055766ede56cdd8246d3532891](https://github.com/actualdev-here/Zinc/commit/6baeeb0025b264055766ede56cdd8246d3532891),
directory `packages/zinc-discord`. Use the compiler and native networking runtime
built from that snapshot. Linux/LLVM with libcurl WebSockets and libcrypto is
required for connected clients. This repository does not distribute a compiler.

Published manifest changes: sibling-path dependencies are replaced by immutable
`v0.1.0` HTTPS Git sources for zinc-json and zinc-net. Source tests remain in
Zinc: `make discord-package-check discord-network-check` covers 453 protocol,
354 permission-bitset differential and 306 JSON cases, plus local TLS/WSS and
connected-client tests. These counts do not measure feature parity or qualify
long-running production memory behavior. No real Discord token was used.

Single-shard, bounded, cooperative and process-local: see README and the source
roadmap for unsupported features, limits and lifecycle requirements.

License: MIT, see LICENSE.
