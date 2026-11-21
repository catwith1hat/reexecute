# Pre-BAL Re-Execution Experiment (`reexecute`)

This repository coordinates the **Pre-Execution Block Access List (Pre-BAL)** experiment across the Ethereum Execution API specification (`execution-apis`) and six execution-layer (EL) clients (`besu`, `erigon`, `ethrex`, `go-ethereum`, `nethermind`, and `reth`).

See [SPEC.md](./SPEC.md) for the full architectural and wire-format specification, and [execution-apis/src/engine/pre-bal.md](./execution-apis/src/engine/pre-bal.md) for the Engine API specification.

## Experiment Summary

Unlike Ethereum's adopted **EIP-7928 Block Access List (BAL)**—which records *post-transaction* state mutations and read-only slot keys to accelerate parallel disk prefetching on stateful nodes—this experiment introduces a **Pre-BAL**:

1. **Producer Mode (`produce_pre_bal = true` on `engine_newPayloadV*`)**:
   A stateful EL (**Besu** first) executes and validates an incoming `ExecutionPayload` normally (persisting the non-canonical block/state as usual until `engine_forkchoiceUpdated`) and additionally returns a hex-encoded RLP `pre_bal` containing the exact **pre-execution** account states, bytecodes, read/written storage slot values, RLP-encoded parent header, and accessed `BLOCKHASH` values.
2. **Consumer Mode (`pre_bal` supplied on `engine_newPayloadV*`)**:
   A state-free EL (**Erigon**, **Reth**, **ethrex**, **Geth**, **Nethermind**) receives the `ExecutionPayload` together with the hex-encoded RLP `pre_bal`, determines the active fork rules from the payload's `blockNumber` and `timestamp`, populates an isolated closed-world in-memory **Fake Store** where *only* the supplied `pre_bal` entries can be read, enforces **exact match** (both missing reads and unused `pre_bal` entries fail validation), skips Merkle Patricia Trie `stateRoot` computation, and performs pure stateless verification (`gasUsed`, `blobGasUsed`, `receiptsRoot`, `logsBloom`, `executionRequests`, and EIP-7928 `blockAccessList` if applicable) with zero persistence.

## Repository Layout

Each component lives in a shallow Git submodule pinned by commits in the root repository:

| Submodule | Upstream | Role in Experiment |
| :--- | :--- | :--- |
| `execution-apis` | `https://github.com/ethereum/execution-apis.git` | Canonical Engine API & OpenRPC specification (`src/engine/pre-bal.md`, `src/schemas/pre-bal.yaml`) |
| `besu` | `https://github.com/besu-eth/besu.git` | Java EL client — **Pre-BAL Producer** |
| `erigon` | `https://github.com/erigontech/erigon.git` | Go EL client — **Pre-BAL Consumer** |
| `ethrex` | `https://github.com/lambdaclass/ethrex.git` | Rust EL client (LambdaClass) — **Pre-BAL Consumer** |
| `go-ethereum` | `https://github.com/ethereum/go-ethereum.git` | Go EL client (Geth) — **Pre-BAL Consumer** |
| `nethermind` | `https://github.com/NethermindEth/nethermind.git` | C# / .NET EL client — **Pre-BAL Consumer** |
| `reth` | `https://github.com/paradigmxyz/reth.git` | Rust EL client (Paradigm) — **Pre-BAL Consumer** |
| `vouch` | `https://github.com/attestantio/vouch.git` | Go Engine API verification proxy — **Pre-BAL Coordinator / Proxy** |

## Clone

```sh
git clone <repo-url> reexecute
cd reexecute
git submodule update --init --jobs 6
```

Submodule checkouts are shallow by default. If a client's build or tests require nested submodules, initialize them from inside that client directory with:

```sh
git submodule update --init --recursive --depth 1
```

## Refresh Submodules (Baseline Update Only)

Save any client or `execution-apis` work on named branches before refreshing, as `git submodule update --remote` leaves detached `HEAD` checkouts:

```sh
git submodule sync --recursive
git submodule update --init --remote --depth 1 --jobs 6
git submodule status
```

## Working Guidelines across Submodules

- **Spec First**: Keep `SPEC.md` and `execution-apis` (`src/engine/pre-bal.md`, `src/schemas/pre-bal.yaml`, `src/engine/openrpc/`) synchronized so all six clients implement the exact same RLP wire format and exact-match access semantics.
- **Client Guidelines**: Respect each client's local agent/contributor instructions (`erigon/AGENTS.md`, `go-ethereum/AGENTS.md`, `reth/AGENTS.md`, `nethermind/AGENTS.md`, `besu/CONTRIBUTING.md`, `ethrex/CONTRIBUTING.md`).
- **Cross-Client Compatibility**: Use identical payloads, chain configurations, and hex-encoded RLP `PreBlockAccessList` fixtures across all clients so Besu's Producer output is consumed directly by Erigon, Reth, ethrex, Geth, and Nethermind.
