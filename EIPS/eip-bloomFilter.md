---
eip: <to be assigned>
title: Optional Bloom Filter for eth_getLogs
description: Adds an optional `bloomFilter` field to the `eth_getLogs` filter object so callers can hide the exact `address`/`topics` they are searching for.
author: Simon Jentzsch (@simon-jentzsch)
discussions-to: https://ethereum-magicians.org/t/<to-be-created>
status: Draft
type: Standards Track
category: Interface
created: 2026-07-02
---

## Abstract

This EIP adds an optional `bloomFilter` field to the filter object of `eth_getLogs` (and, by reference, `eth_newFilter`). `bloomFilter` is an array of 2048-bit Ethereum log blooms. When `bloomFilter` is present, the server returns every log whose own bloom is a bit-superset of at least one array entry; when `address` and/or `topics` are also present, the server additionally applies their standard `eth_getLogs` semantics as an `AND` filter. The response format is unchanged. The caller then applies its exact filter locally to remove Bloom false positives.

The extension lets clients probe for logs without revealing the exact set of addresses or topic values they care about: only a probabilistic, lossy representation of the query is sent on the wire, and the server returns a superset of the true matches.

## Motivation

`eth_getLogs` today accepts only concrete values for `address` and `topics`. Any caller that queries for events involving its own address (`Transfer` recipients, allowance grants, protocol interactions, ...) therefore hands the exact interest set to whichever RPC provider serves the request. For light clients, wallets and privacy-preserving stacks this is one of the strongest content-level identity/intent leaks the JSON-RPC surface still exposes.

At the same time every execution client already indexes `logsBloom` per block precisely to make `eth_getLogs` cheap: the range scan is a Bloom-subset test over block headers. Exposing that same primitive at the RPC boundary requires no new indexing and keeps the request-side data structure identical to the one the client already computes internally.

By letting the caller send one or more pre-computed blooms instead of concrete filter values:

- The provider sees a probabilistic query and cannot recover the exact `address`/`topics` the caller is interested in (subject to the security considerations in this document).
- The wire request stays small (a few hundred bytes per bloom, up to a bounded number of blooms).
- Existing server-side infrastructure for range scans can be reused verbatim as a prefilter (a block whose `logsBloom` matches no entry cannot contain a matching log); the per-log check reuses the same 256-byte bit-subset primitive.
- The result format is unchanged, so existing client-side JSON parsers keep working.

The extension composes with, but does not depend on, cryptographic verification schemes such as verifiable RPC responses: a `bloomFilter` query returns a superset that the client can independently verify (for integrity) or accept on trust (for the classic JSON-RPC transport). It is deliberately transport-agnostic.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Filter object extension

The filter object accepted by `eth_getLogs` (and `eth_newFilter`) is extended with one optional field:

| Field | Type | Description |
|---|---|---|
| `bloomFilter` | `Array` of `DATA` (each exactly 256 bytes, i.e. 512 hex characters after the `0x` prefix) | Set of candidate 2048-bit log blooms. |

The remaining filter fields (`fromBlock`, `toBlock`, `blockHash`, `address`, `topics`) keep their existing semantics from the JSON-RPC specification.

A server MAY advertise support for `bloomFilter` through the `rpc_modules` / `web3_clientVersion` mechanism it already uses for other extensions; discovery is out of scope for this EIP.

### Bloom encoding

Each `bloomFilter` entry MUST be exactly 256 bytes (2048 bits). Its byte and bit ordering are identical to the `logsBloom` field of execution-layer block headers, i.e. the M3:2048 bloom defined by the Ethereum Yellow Paper. Formally, a bit index `i` in the range `[0, 2048)` maps to:

```
byte_index = 255 - ((i >> 3) & 0xff)
bit_mask   = 1 << (i & 7)
```

This is the identical mapping used to compute an execution-layer block's `logsBloom` from its logs, which lets a server test "entry is a subset of block bloom" as a plain bitwise `AND` comparison over 256 bytes.

The intended construction (informative, not normative):

- For a single log with address `A` and topics `T0..Tk`, the log's own bloom is the union of `M3(A)`, `M3(T0)`, ..., `M3(Tk)`, where `M3(x)` sets three bits at positions taken from bytes `(0,1)`, `(2,3)`, `(4,5)` of `keccak256(x)` masked with `0x7ff` (11 bits each). This is the same computation the execution layer applies when it builds a block's `logsBloom`.
- A `bloomFilter` entry SHOULD therefore be constructed as the union of `M3(...)` for exactly one candidate address (or none) and one candidate topic value per topic position. Callers that want to query several combinations MUST send one array entry per combination.
- Callers MAY clear (unset) additional bits from a computed entry to broaden the match set and increase the amount of noise returned. Callers MUST NOT set bits that were not produced by the construction above unless they are willing to accept that the entry will produce no matches beyond the ones that entry alone would already have produced (adding bits can only ever reduce the match set).

### Matching semantics

For any log `L` on chain, define its **log bloom** as

```
log_bloom(L) = M3(L.address) | M3(L.topics[0]) | M3(L.topics[1]) | ... | M3(L.topics[k-1])
```

where `M3(x)` is the 2048-bit Ethereum bloom mapping (see [Bloom encoding](#bloom-encoding)). This is the identical construction the execution layer applies when it builds a block's `logsBloom` from that block's logs; consequently `log_bloom(L)` is by construction a bit-subset of `B.logsBloom` for the block `B` containing `L`.

Let `bf` be the value of `bloomFilter` in the request. When `bf` is present and non-empty, a server implementing this EIP MUST evaluate every log `L` in the requested scope (the log range identified by `fromBlock`/`toBlock`, or the block identified by `blockHash`) as follows:

1. Compute `bloom_match = false`.
2. For each entry `e` in `bf`: if `(e AND log_bloom(L)) == e` (i.e. `e` is a bit-subset of `L`'s own bloom), set `bloom_match = true` and stop iterating over entries.
3. If `bloom_match` is `false`, the server MUST NOT include `L`.
4. Otherwise the server MUST additionally evaluate `address` and `topics` against `L` following the unchanged `eth_getLogs` semantics: `L` MUST be included iff (a) `address` is absent or `L.address` matches it, AND (b) for each present entry of `topics`, the corresponding `L.topics[i]` matches that entry (`null` = wildcard, string = equality, array = OR). If either check fails, the server MUST NOT include `L`.
5. Logs that pass both checks MUST appear in the response in the canonical order defined by the existing `eth_getLogs` semantics.

The result set is therefore the set of all logs whose own bloom is a superset of at least one entry in `bf` AND that satisfy the (optional) `address`/`topics` constraints. This is by construction a superset of the caller's true intent up to the false-positive rate of the Bloom encoding.

A server MAY use the per-block `logsBloom` field as an internal prefilter (a block whose `logsBloom` is not a superset of any `bf` entry cannot contain a matching log), but such an optimisation MUST NOT change the observable result: the final response MUST be filtered at log granularity as defined above.

### Interaction with `address` and `topics`

`bloomFilter`, `address` and `topics` are independent filter fields; when more than one is present the server MUST apply their intersection (`AND`), as specified in [Matching semantics](#matching-semantics). `address` and `topics` retain their unmodified `eth_getLogs` semantics; `bloomFilter` layers on top as a further constraint on `log_bloom(L)`.

- Callers that value privacy over server-side efficiency SHOULD omit `address` and `topics` when they send `bloomFilter`. `address` and `topics` are transmitted verbatim; sending them defeats the privacy purpose of `bloomFilter` for exactly those fields.
- Callers that use `bloomFilter` to reduce server work or bandwidth (e.g. when the caller is a trusted party or is willing to expose the exact filter) MAY send all three fields; the server MUST honour the intersection.

### Response

The response is a JSON array of log objects in the same schema as unmodified `eth_getLogs`. No new fields are introduced. Clients MUST filter the returned array locally against their true `(address, topics)` interest before delivering it to the calling application.

### Error handling

A server implementing this EIP MUST reject a request with JSON-RPC error code `-32602` ("Invalid params") if:

- `bloomFilter` is present but is not a JSON array;
- `bloomFilter` is present and is an empty array;
- any entry of `bloomFilter` is not a `DATA` hex string that decodes to exactly 256 bytes.

Servers MAY additionally impose a maximum number of entries and reject requests exceeding it with the same error code. Sixteen entries per request is RECOMMENDED as a floor; servers MAY accept more.

### `eth_newFilter`

`eth_newFilter` accepts the same filter object as `eth_getLogs`. A server that supports `bloomFilter` for `eth_getLogs` SHOULD also accept it in `eth_newFilter`; subsequent calls to `eth_getFilterChanges` and `eth_getFilterLogs` for that filter MUST use the matching semantics defined above.

## Rationale

### Why a Bloom filter, and why 2048 bits

Every execution-layer block header already carries a 2048-bit `logsBloom` computed by the M3:2048 function. Reusing the same encoding at the RPC boundary is the least invasive extension possible: the server's range scan already tests block blooms against a filter bloom to decide whether to inspect a block, and the new field simply exposes that bloom directly. A different-sized bloom would require an additional projection and would break the direct-subset test.

### Why an array with OR semantics

A single 2048-bit bloom cannot represent multiple disjunctive candidates without conflating them into a broader match set. For a caller interested in `Transfer` events from any of five contracts, a single merged bloom would set the union of all their bits and match many more blocks than necessary. An explicit array with OR semantics lets the caller keep each candidate combination sharp while still expressing "any of these".

### Why log granularity

Two designs were considered:

- **Log granularity (chosen)**: the server tests each log's own bloom against `bloomFilter` and returns only logs whose bloom is a superset of at least one entry.
- Block granularity: the server would return every log of every block whose `logsBloom` matches at least one entry.

Log granularity is strictly more efficient on the wire: the caller-side Bloom false-positive rate is set by how many bits the caller chose to leave set in each entry, and applies to individual logs (~4 elements set per entry) rather than to whole blocks whose `logsBloom` unions hundreds of logs. In practice a well-constructed entry causes very few log-level false positives whereas at block level it will match many busy blocks. Block granularity would inflate the response and, in return, offer no additional privacy benefit beyond what the Bloom encoding already guarantees: when the caller uses `bloomFilter` for privacy (and therefore omits `address`/`topics` per [Interaction with `address` and `topics`](#interaction-with-address-and-topics)), the exact `(address, topics)` set is never on the wire in either mode. Any additional metadata leakage from log-level responses (which specific logs came back) is already contained in the fact that the caller retrieves those logs.

A server may still use the per-block `logsBloom` as an internal prefilter, but the observable output has to be log-granular; the normative rules are stated in [Matching semantics](#matching-semantics).

### Why the caller filters locally

Bloom membership is probabilistic and one-way: even a perfectly constructed entry produces occasional false-positive log matches (a log whose `(address, topics)` is not what the caller wants, but whose Bloom happens to be a superset of the entry due to keccak collisions). A server implementing this EIP cannot know which of the returned logs the caller was actually looking for, so the final refinement must happen on the client. Because the caller retains its exact `(address, topics)` interest locally, this refinement is a straightforward linear pass over the response.

### Why transport-agnostic

`bloomFilter` is orthogonal to how the response is transported or authenticated. The same request works over plain HTTPS JSON-RPC (unverified), over verifiable-RPC transports that additionally carry a cryptographic proof of inclusion for every log, and over any future transport that ships JSON-RPC results with metadata. Making the field part of the standard filter object rather than a separate method keeps that composability.

## Backwards Compatibility

The extension is opt-in on both sides of the connection and preserves the pre-existing `eth_getLogs` semantics in full. Concretely:

- Servers that do not implement this EIP receive an unknown extra property on the filter object. JSON-RPC servers already tolerate unknown properties on `eth_getLogs` filter objects and treat them as no-ops; combined with the recommendation to omit `address` and `topics` when using `bloomFilter`, such a legacy server will simply return every log in the requested range. That is still a strict superset of the caller's real interest, so the client-side refinement produces the correct final set; only the bandwidth cost increases.
- Clients that do not implement this EIP send filters exactly as today and are unaffected.
- No consensus rules and no existing JSON-RPC semantics change: the extension is a strict addition to the filter object.

## Test Cases

### 1. Filter object with a single bloom entry

Client-side interest: `Transfer(to = 0x4838b106fce9647bdf1e7877bf73ce8b0bad5f97)` events emitted by the token contract `0x73f7b1184B5cD361cC0f7654998953E2a251dd58`, restricted to block `0x18446ed`.

Real filter (never sent when `bloomFilter` is used):

```json
{
  "fromBlock": "0x18446ed",
  "toBlock":   "0x18446ed",
  "address":   "0x73f7b1184B5cD361cC0f7654998953E2a251dd58",
  "topics": [
    "0x85177f287940f2f05425a4029951af0e047a7f9c4eaa9a6e6917bcd869f86695",
    "0x0000000000000000000000004838b106fce9647bdf1e7877bf73ce8b0bad5f97"
  ]
}
```

The single bloom entry is `M3(address) | M3(topic0) | M3(topic1)`. Wire request:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "eth_getLogs",
  "params": [{
    "fromBlock": "0x18446ed",
    "toBlock":   "0x18446ed",
    "bloomFilter": [
      "0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
    ]
  }]
}
```

A compliant server returns every log in that block whose own bloom is a superset of the entry above. In the concrete transaction above this is the single `Transfer` log emitted by the token contract; any unrelated log whose Bloom happens to be a superset of the entry (Bloom false positive) would also be returned. The client discards every returned log whose `address`, `topic0`, `topic1` do not match its real filter.

### 2. Multiple candidates (OR)

Client-side interest: `Transfer` events from *either* USDC or DAI (any recipient) in a small block range.

Wire request:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "eth_getLogs",
  "params": [{
    "fromBlock": "0x1000",
    "toBlock":   "0x1005",
    "bloomFilter": [
      "0x<M3(USDC) | M3(Transfer_sig), 256 bytes hex>",
      "0x<M3(DAI)  | M3(Transfer_sig), 256 bytes hex>"
    ]
  }]
}
```

A log in the range is included in the response whenever its own bloom is a superset of at least one entry.

### 3. Malformed requests

The following requests are rejected with error code `-32602`:

- `"bloomFilter": []`
- `"bloomFilter": "0x00...(256 bytes)"` (not an array)
- `"bloomFilter": ["0x1234"]` (entry is not 256 bytes)

## Reference Implementation

A reference client and server implementing this EIP are published under the MIT license as part of [corpus-core/colibri-stateless](https://github.com/corpus-core/colibri-stateless). The relevant files are:

- Bloom construction (client side): `src/chains/eth/verifier/eth_bloom.c` (`c4_eth_create_bloomfilter`, `build_bloom_variants`) — computes one M3:2048 entry per candidate `(address, topics)` combination.
- Request rewriting: `src/chains/eth/verifier/eth_verify.c` (`c4_eth_get_prover_payload`) — strips `address`/`topics` from the filter object and substitutes `bloomFilter` before sending the request.
- Server-side parsing and subset match: `src/chains/eth/prover/logs_cache.c` (`parse_bloom_filter_array`, `bloom_subset_of64`, `bloom_matches`) — decodes each entry as exactly 256 bytes and provides the `(entry AND target) == entry` primitive that this EIP's matching rule is expressed in.
- Log-granular refinement: `src/chains/eth/verifier/eth_bloom.c` (`c4_eth_filter_logs`) — implements the mandatory local re-filtering against the caller's exact `(address, topics)` interest.

Internally the reference server currently performs the range scan block-granularly (via `logs_cache.c::build_match_index`, which uses the per-block `logsBloom` as a prefilter and then delivers all logs of matching blocks) and produces the log-granular result mandated by this EIP through the downstream refinement step in `c4_eth_filter_logs`. Moving the log-granular filter into the server itself is a straightforward per-log application of `bloom_matches` against `log_bloom(L)` and does not change the wire format. The reference client currently strips `address` and `topics` from the outgoing filter (for privacy); a strict EIP-conformant server implementation additionally needs to honour those fields as an `AND` filter when they are present, which is a purely additive change.

## Security Considerations

### The privacy guarantee is probabilistic, not cryptographic

`bloomFilter` hides the caller's exact interest behind a lossy projection. It does not provide anonymity. In particular:

- The bloom encoding maps every element (address or topic) to only three bits out of 2048, and the set of high-value elements an RPC caller might look up (major token contracts, popular event signatures, well-known addresses) is small and public. An adversary that maintains a precomputed dictionary of `M3(x)` for every plausible `x` and takes the intersection over many requests can often recover the actually queried set.
- Callers that need stronger guarantees are advised to combine this extension with transport-level measures (rotating providers, Tor, etc.); on its own this EIP only raises the cost of profiling rather than making it impossible.

### False negatives are impossible; false positives are inherent

A `bloomFilter` entry `e` constructed as `M3(A) | M3(T0) | ...` for a specific candidate `(address, topics)` combination is by construction a bit-subset of `log_bloom(L)` for every log `L` whose `(address, topics)` equals that combination — the entry contains exactly the bits that `M3` would set for those very elements, and `log_bloom(L)` contains those same bits plus possibly others. A correctly constructed entry therefore cannot cause the server to miss such a log. False positives (logs whose Bloom happens to be a superset of an entry through keccak collisions on unrelated `(address, topics)` values) are inherent to the Bloom encoding and are the mechanism that produces the privacy-relevant noise; clients handle them via the mandatory local re-filtering step.

### Bandwidth and denial of service

Servers are expected to enforce their existing response-size, block-range, and rate-limit policies unchanged; those policies already bound the worst case of `eth_getLogs`. Implementations should avoid allocating memory sized by the number of `bloomFilter` entries before validating that each entry decodes to exactly 256 bytes. A server may use per-block `logsBloom` as an internal prefilter to skip blocks that cannot contain any matching log; this is a pure optimisation and does not change the observable log-granular result required by [Matching semantics](#matching-semantics).

### Integrity is out of scope

`eth_getLogs` today returns unauthenticated JSON, and this EIP does not change that. A malicious server can still omit matching blocks, return fabricated logs, or lie about the block's `logsBloom`. Clients that require integrity are expected to verify responses against a cryptographic proof anchored to consensus (for example the mechanism described by a companion verifiable-RPC EIP).

## Privacy Considerations

Beyond the probabilistic guarantee discussed above, callers using this extension should be aware of two secondary leaks:

- **Intersection across requests.** When the same real filter is used to poll for new logs over time, freshly generated `bloomFilter` values would each contain the true bits plus different noise. An observer of multiple polls can intersect them and recover the constant (real) bits. Callers polling the same interest are advised to reuse the same `bloomFilter` value across polls so that no intersection reveals more than a single request would.
- **Block-range choice.** `fromBlock` and `toBlock` are still visible on the wire and can themselves be an identity/intent signal (for example a very narrow range around a specific block). This EIP does not address that channel.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE).
