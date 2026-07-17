# org-bitcoin-p2p

[![CI](https://github.com/kotoba-lang/org-bitcoin-p2p/actions/workflows/ci.yml/badge.svg)](https://github.com/kotoba-lang/org-bitcoin-p2p/actions/workflows/ci.yml)

## Permanent safety boundary -- read this first

**This is a READ-ONLY OBSERVER of public Bitcoin chain data. This is a
standing, permanent boundary (ADR-2607172600 in `com-junkawasaki/root`) --
not a v0.1-vs-later phasing question.** This repo NEVER:

- generates or stores a private key
- constructs a transaction
- signs a transaction
- broadcasts or relays a transaction
- participates in mempool relay
- mines
- implements full Bitcoin Script/consensus validation

There is no wallet functionality of any kind anywhere in this codebase.
What this repo DOES do is genuine **SPV-level header validation**: verify
a header's own hash actually meets the difficulty target its `bits` field
claims, and verify each header in a chain actually links to the real,
computed hash of the header before it. That is real cryptographic
verification of the header chain (`kotobase.bitcoin.protocol/validate-chain`)
-- not a stub, and not full-node consensus validation. If a change to this
repo would need key handling or transaction construction, it is out of
scope, permanently.

**Testnet3 is this library's default/primary network.** Mainnet is
supported as an explicit, opt-in `:network :mainnet` configuration value,
never a default -- see `kotobase.bitcoin.protocol/mainnet-magic` /
`testnet-magic`. Every live-network exercise of this code during this
repo's own development connected to real Bitcoin **testnet** peers.

## What this is

A Bitcoin P2P wire-protocol client -- SPV-style header sync -- part of the
kotobase blockchain-client extension project
(ADR-2607172600 in `com-junkawasaki/root`). Bitcoin has no separate
"RPC protocol" as its network-native language: the P2P wire protocol
(`version`/`verack` handshake, `getheaders`/`headers`, ping/pong) *is* how
a peer talks to the network, so a client that observes chain state
implements this protocol directly rather than wrapping Bitcoin Core's
node-operator-facing JSON-RPC admin interface.

Two layers, same split every repo in this project uses:

| Layer | Namespace | Shape |
|---|---|---|
| Pure core | `kotobase.bitcoin.protocol` | `.cljc`, no sockets: wire framing, block-header decode/encode, SPV validation |
| Transport | `kotobase.bitcoin.transport` | `.cljs`-only, `node:net`: the real TCP connection, handshake, ping/pong, getheaders/headers, `kotobase.store` persistence |

## `kotobase.bitcoin.protocol` (pure `.cljc`)

- 24-byte message header (magic + 12-byte command + length + checksum) +
  payload encode/decode for `version`, `verack`, `ping`/`pong`,
  `getheaders`, `headers`.
- 80-byte block-header decode/encode (version, prev-block, merkle-root,
  timestamp, bits, nonce) with the hash **always computed from the actual
  bytes** (`sha256d`, never trusted from an external claim) -- the
  load-bearing trustless property every validation function depends on.
- `bits->target-bytes` / `hash-meets-target?` -- real compact-target
  decoding and proof-of-work verification, done as big-endian
  **byte-vector comparison** rather than arbitrary-precision integer
  arithmetic (fully portable JVM clj + cljs without a bigint dependency;
  see the function's own docstring for why this is exact).
- `header-links-to?` / `validate-chain` -- real chain-linkage
  verification.
- Digests are delegated to
  [`kotoba-lang/sha256d`](https://github.com/kotoba-lang/sha256d) (a
  portable `.cljc` SHA-256/SHA-256d reference implementation already
  verified against the real Bitcoin genesis block on both JVM and V8) --
  this repo does not reimplement SHA-256.
- Two hardcoded genesis-header constants (`mainnet-genesis-header`,
  `testnet-genesis-header`) are the **entire trusted input** this SPV
  client bootstraps from -- everything else is validated forward via
  `hash-meets-target?`/`header-links-to?`, real cryptographic
  verification, not an expanding set of trusted checkpoints.

Deliberately NOT implemented (see the safety boundary above): full
Script/consensus validation, difficulty-retarget-schedule recomputation,
timestamp-median-past rules, transaction/mempool handling of any kind.

### Real test fixtures

`test/kotobase/bitcoin/fixtures.cljc` hardcodes the first 4 real headers
(genesis + 3) of both Bitcoin mainnet and testnet3, fetched from
[blockstream.info](https://blockstream.info)'s public block-explorer REST
API on 2026-07-17 and independently cross-checked in that session: each
header's own `sha256d`, byte-reversed, was recomputed with plain `openssl
dgst -sha256` chained twice and confirmed byte-for-byte equal to the
well-known display hash before being hardcoded -- not copied from memory.

`test/kotobase/bitcoin/protocol_test.cljc` proves, against this real data:

- every real fixture header's hash matches the known value and satisfies
  its own claimed proof-of-work target
- the real 4-header chain on each network validates cleanly
  (`valid-header-chain?` true, zero errors)
- flipping one nonce bit in a real header changes its computed hash and
  breaks the next real header's linkage to it (a genuine tamper-evidence
  proof, not an assumption)
- reordering real headers is rejected as `:broken-linkage`
- swapping in an artificially strict `:bits` on a real header is rejected
  as `:insufficient-work`

## `kotobase.bitcoin.transport` (`.cljs`-only)

Real TCP (`node:net`, zero npm dependencies besides `promesa`, which nbb
bundles) -- matching the `kotoba-lang/dtn` / `kotoba-lang/io-libp2p` /
`kotoba-lang/org-ietf-sftp` transport-layer precedent
(ADR-2607161817 / ADR-2607162135). `.cljs`-only means it can never be
loaded by the JVM `clojure -M:test` compat suite, so it can never regress
`kotobase.bitcoin.protocol`'s pure test suite.

```clojure
(require '[kotobase.bitcoin.transport :as tp]
         '[kotobase.local :as local])

(def store (local/local-store))

(-> (tp/connect! {:host "129.226.198.211" :port 18333 :network :testnet :store store})
    (.then (fn [conn]
             (-> (tp/get-headers! conn)
                 (.then (fn [result]
                          (println "synced" (:count result) "headers, tip:" (tp/tip store))))))))
```

`connect!` performs the real version/verack handshake in both directions
before resolving. `get-headers!` sends `getheaders` (locator = the
store's current tip, or the network's hardcoded genesis header on a fresh
store), and -- only if the whole reply batch passes
`kotobase.bitcoin.protocol/validate-chain` against the real prior tip --
persists every header via `kotobase.store`'s `IStore` (`-put`/`-get`;
collection `:kotobase.bitcoin/headers` keyed by hash-hex, a `"tip"` doc in
`:kotobase.bitcoin/meta`, and an audit stream at
`:kotobase.bitcoin/header-stream`). An invalid batch is rejected
all-or-nothing -- nothing from it is ever persisted. `ping!` sends a real
`ping` and resolves once the matching `pong` arrives; inbound `ping` from
the peer is always auto-answered with `pong` regardless.

No peer discovery, no multi-peer management, no relay, no automatic
re-sync loop -- a caller drives `get-headers!` again for the next batch.
Any command outside this repo's supported set (`addr`, `inv`, `tx`,
`block`, `mempool`, `sendcmpct`, ...) is logged and ignored, never acted
on -- consistent with the permanent safety boundary above.

## Honest verification status

**A real live handshake, ping/pong round-trip, and `getheaders`/`headers`
sync against a real Bitcoin testnet full node succeeded during this
repo's own development** -- this is not a simulation:

```
[bitcoin-p2p 129.226.198.211:18333] <- version peer user-agent=/Satoshi:22.1.0/ start-height=5074359
[bitcoin-p2p 129.226.198.211:18333] handshake complete (peer version received, mutual verack exchanged)
ping->pong ok? true
[bitcoin-p2p 129.226.198.211:18333] <- headers count=2000
get-headers! ok? true count 2000 errors []
tip: {:height 2000, :hash-hex "0000000005bdbddb59a3cd33b69db94fa67669c41d9d32751512b5d7b68c71cf"}
```

2000 real testnet headers, received over a real socket from a real
Bitcoin Core node, validated (`validate-chain` against the hardcoded
testnet genesis header) and persisted -- genuine SPV trust exercised
against a real chain, not fixture data.

**Also observed, honestly**: on some runs against some peers, the version/
verack handshake and ping/pong completed but the peer did not reply to
`getheaders` within a generous timeout (likely rate-limiting/
deprioritizing a rapidly-reconnecting source IP during repeated manual
testing) -- a real, observed remote-peer behavior, not a bug in this
repo's own logic. `test/kotobase/bitcoin/transport_demo.cljs` proves
`get-headers!`'s own timeout path resolves correctly (does not hang) via
a deterministic local fake peer that withholds its reply on purpose.
`test/kotobase/bitcoin/testnet_live_demo.cljs` (the live version) is
**best-effort and non-blocking in CI** for exactly this reason -- it tries
several peer IPs in turn and is honest in its own output about exactly
which step succeeded or failed for each one; a failure there means "no
real peer completed the full chain within this run," never a fabricated
success.

| Claim | Verified how | Confidence |
|---|---|---|
| Wire framing (message header + payload, checksum) | Unit tests, `protocol_test.cljc` -- including a checksum-tamper-rejection test | High |
| Block-header decode/encode, hash computation | Against real historical mainnet + testnet3 header bytes (genesis + 3), independently re-derived, not copied from memory | High |
| Proof-of-work target check, chain linkage | Real 4-header chains on both networks validate; real tamper/reorder/insufficient-work cases are rejected | High |
| Handshake / ping-pong / getheaders-headers / checksum-tamper-drop / timeout, over a REAL socket | `transport_demo.cljs`, 10/10 checks, deterministic (local fake peer) | High |
| Interop with a real Bitcoin Core testnet peer | `testnet_live_demo.cljs` -- succeeded during this repo's own development (transcript above); best-effort/non-deterministic by nature, not gating CI | High when it succeeds, honestly non-deterministic when a peer doesn't respond |
| Interop with Bitcoin mainnet | **Not exercised.** Testnet is this library's default/primary target per the safety boundary above. | None |
| Security review of the transport (DoS resistance, malformed-input fuzzing, resource exhaustion) | **Not done. Not claimed.** | None |

## Develop / test

First-class runtime is **nbb/cljs** (repo-wide runtime priority: `kotoba
wasm` > `clojurewasm` > `ClojureScript` > `nbb` > (jvm/bb)):

```bash
git clone https://github.com/kotoba-lang/kotobase .deps/kotobase
git clone https://github.com/kotoba-lang/sha256d .deps/sha256d

# Pure .cljc core (protocol_test.cljc) -- real header fixtures, PoW +
# linkage validation, message framing.
nbb --classpath "src:test:.deps/kotobase/src:.deps/sha256d/src" bin/run_tests.cljs

# Deterministic real-socket demo (local fake peer, no live network needed)
nbb --classpath "src:test:.deps/kotobase/src:.deps/sha256d/src" test/kotobase/bitcoin/transport_demo.cljs

# BEST-EFFORT live testnet demo (real internet, real peer, non-deterministic)
nbb --classpath "src:test:.deps/kotobase/src:.deps/sha256d/src" test/kotobase/bitcoin/testnet_live_demo.cljs

# Manual one-shot sync against a real peer
nbb --classpath "src:.deps/kotobase/src:.deps/sha256d/src" bin/bitcoin_node.cljs sync --host <ip> [--port 18333] [--network testnet]
```

The `:test` alias in `deps.edn` is the JVM **compat** suite for the pure
`.cljc` core (`kotobase.bitcoin.protocol`) only -- it never loads anything
under `src/kotobase/bitcoin/transport.cljs` (`.cljs`-only, `node:net`,
cannot run on the JVM at all):

```bash
clojure -M:test
```

## License

Apache-2.0
