# Upgrading Tendermint Core

This guide provides instructions for upgrading to specific versions of
Tendermint Core.

## Unreleased

## Config Changes

* A new config field, `BootstrapPeers` has been introduced as a means of
  adding a list of addresses to the addressbook upon initializing a node. This is an
  alternative to `PersistentPeers`. `PersistentPeers` shold be only used for
  nodes that you want to keep a constant connection with i.e. sentry nodes

----

### ABCI Changes

* The `ABCIVersion` is now `1.0.0`.

* Added new ABCI methods `PrepareProposal` and `ProcessProposal`. For details,
  please see the [spec](spec/abci/README.md). Applications upgrading to
  v0.37.0 must implement these methods, at the very minimum, as described
  [here](./spec/abci/abci++_app_requirements.md)
* Deduplicated `ConsensusParams` and `BlockParams`.
  In the v0.34 branch they are defined both in `abci/types.proto` and `types/params.proto`.
  The definitions in `abci/types.proto` have been removed.
  In-process applications should make sure they are not using the deleted
  version of those structures.
* In v0.34, messages on the wire used to be length-delimited with `int64` varint
  values, which was inconsistent with the `uint64` varint length delimiters used
  in the P2P layer. Both now consistently use `uint64` varint length delimiters.
* Added `AbciVersion` to `RequestInfo`.
  Applications should check that Tendermint's ABCI version matches the one they expect
  in order to ensure compatibility.
* The `SetOption` method has been removed from the ABCI `Client` interface.
  The corresponding Protobuf types have been deprecated.
* The `key` and `value` fields in the `EventAttribute` type have been changed
  from type `bytes` to `string`. As per the [Protocol Buffers updating
  guidelines](https://developers.google.com/protocol-buffers/docs/proto3#updating),
  this should have no effect on the wire-level encoding for UTF8-encoded
  strings.

## v0.34.20

### Feature: Priority Mempool

This release backports an implementation of the Priority Mempool from the v0.35
branch. This implementation of the mempool permits the application to set a
priority on each transaction during CheckTx, and during block selection the
highest-priority transactions are chosen (subject to the constraints on size
and gas cost).

Operators can enable the priority mempool by setting `mempool.version` to
`"v1"` in the `config.toml`. For more technical details about the priority
mempool, see [ADR 067: Mempool
Refactor](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/main/docs/architecture/adr-067-mempool-refactor.md).

## v0.34.0

**Upgrading to Tendermint 0.34 requires a blockchain restart.**
This release is not compatible with previous blockchains due to changes to
the encoding format (see "Protocol Buffers," below) and the block header (see "Blockchain Protocol").

Note also that Tendermint 0.34 also requires Go 1.16 or higher.

### ABCI Changes

* The `ABCIVersion` is now `0.17.0`.

* New ABCI methods (`ListSnapshots`, `LoadSnapshotChunk`, `OfferSnapshot`, and `ApplySnapshotChunk`)
  were added to support the new State Sync feature.
  Previously, syncing a new node to a preexisting network could take days; but with State Sync,
  new nodes are able to join a network in a matter of seconds.
  Read [the spec](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/v0.34.x/spec/abci/apps.md#state-sync)
  if you want to learn more about State Sync, or if you'd like your application to use it.
  (If you don't want to support State Sync in your application, you can just implement these new
  ABCI methods as no-ops, leaving them empty.)

* `KV.Pair` has been replaced with `abci.EventAttribute`. The `EventAttribute.Index` field
  allows ABCI applications to dictate which events should be indexed.

* The blockchain can now start from an arbitrary initial height,
  provided to the application via `RequestInitChain.InitialHeight`.

* ABCI evidence type is now an enum with two recognized types of evidence:
  `DUPLICATE_VOTE` and `LIGHT_CLIENT_ATTACK`.
  Applications should be able to handle these evidence types
  (i.e., through slashing or other accountability measures).

* The [`PublicKey` type](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/main/proto/tendermint/crypto/keys.proto#L13-L15)
  (used in ABCI as part of `ValidatorUpdate`) now uses a `oneof` protobuf type.
  Note that since Tendermint only supports ed25519 validator keys, there's only one
  option in the `oneof`.  For more, see "Protocol Buffers," below.

* The field `Proof`, on the ABCI type `ResponseQuery`, is now named `ProofOps`.
  For more, see "Crypto," below.

### P2P Protocol

The default codec is now proto3, not amino. The schema files can be found in the `/proto`
directory. For more, see "Protobuf," below.

### Blockchain Protocol

* `Header#LastResultsHash`, which is the root hash of a Merkle tree built from
  `ResponseDeliverTx(Code, Data)` as of v0.34 also includes `GasWanted` and `GasUsed`
  fields.

* Merkle hashes of empty trees previously returned nothing, but now return the hash of an empty input,
  to conform with [RFC-6962](https://tools.ietf.org/html/rfc6962).
  This mainly affects `Header#DataHash`, `Header#LastResultsHash`, and
  `Header#EvidenceHash`, which are often empty. Non-empty hashes can also be affected, e.g. if their
  inputs depend on other (empty) Merkle hashes, giving different results.

### Transaction Indexing

Tendermint now relies on the application to tell it which transactions to index. This means that
in the `config.toml`, generated by Tendermint, there is no longer a way to specify which
transactions to index. `tx.height` and `tx.hash` will always be indexed when using the `kv` indexer.

Applications must now choose to either a) enable indexing for all transactions, or
b) allow node operators to decide which transactions to index.
Applications can notify Tendermint to index a specific transaction by setting
`Index: bool` to `true` in the Event Attribute:

```go
[]types.Event{
	{
		Type: "app",
		Attributes: []types.EventAttribute{
			{Key: []byte("creator"), Value: []byte("Cosmoshi Netowoko"), Index: true},
		},
	},
}
```

### Protocol Buffers

Tendermint 0.34 replaces Amino with Protocol Buffers for encoding.
This migration is extensive and results in a number of changes, however,
Tendermint only uses the types generated from Protocol Buffers for disk and
wire serialization.
**This means that these changes should not affect you as a Tendermint user.**

However, Tendermint users and contributors may note the following changes:

* Directory layout changes: All proto files have been moved under one directory, `/proto`.
  This is in line with the recommended file layout by [Buf](https://buf.build).
  For more, see the [Buf documentation](https://buf.build/docs/lint-checkers#file_layout).
* ABCI Changes: As noted in the "ABCI Changes" section above, the `PublicKey` type now uses
  a `oneof` type.

For more on the Protobuf changes, please see our [blog post on this migration](https://medium.com/Cryptocurrency-TOOLS/tendermint-0-34-protocol-buffers-and-you-8c40558939ae).

### Consensus Parameters

Tendermint 0.34 includes new and updated consensus parameters.

#### Version Parameters (New)

* `AppVersion`, which is the version of the ABCI application.

#### Evidence Parameters

* `MaxBytes`, which caps the total amount of evidence. The default is 1048576 (1 MB).

### Crypto

#### Keys

* Keys no longer include a type prefix. For example, ed25519 pubkeys have been renamed from
  `PubKeyEd25519` to `PubKey`. This reduces stutter (e.g., `ed25519.PubKey`).
* Keys are now byte slices (`[]byte`) instead of byte arrays (`[<size>]byte`).
* The multisig functionality that was previously in Tendermint now has
  a new home within the Cosmos SDK:
  [`cosmos/cosmos-sdk/types/multisig`](https://github.com/cosmos/cosmos-sdk/blob/master/crypto/types/multisig/multisignature.go).

#### `merkle` Package

* `SimpleHashFromMap()` and `SimpleProofsFromMap()` were removed.
* The prefix `Simple` has been removed. (For example, `SimpleProof` is now called `Proof`.)
* All protobuf messages have been moved to the `/proto` directory.
* The protobuf message `Proof` that contained multiple ProofOp's has been renamed to `ProofOps`.
  As noted above, this affects the ABCI type `ResponseQuery`:
  The field that was named Proof is now named `ProofOps`.
* `HashFromByteSlices` and `ProofsFromByteSlices` now return a hash for empty inputs, to conform with
  [RFC-6962](https://tools.ietf.org/html/rfc6962).

### `libs` Package

The `bech32` package has moved to the Cosmos SDK:
[`cosmos/cosmos-sdk/types/bech32`](https://github.com/cosmos/cosmos-sdk/tree/4173ea5ebad906dd9b45325bed69b9c655504867/types/bech32).

### CLI

The `tendermint lite` command has been renamed to `tendermint light` and has a slightly different API.
See [the docs](https://docs.tendermint.com/v0.33/tendermint-core/light-client-protocol.html#http-proxy) for details.

### Light Client

We have a new, rewritten light client! You can
[read more](https://medium.com/tendermint/everything-you-need-to-know-about-the-tendermint-light-client-f80d03856f98)
about the justifications and details behind this change.

Other user-relevant changes include:

* The old `lite` package was removed; the new light client uses the `light` package.
* The `Verifier` was broken up into two pieces:
    * Core verification logic (pure `VerifyX` functions)
    * `Client` object, which represents the complete light client
* The new light client stores headers and validator sets as `LightBlock`s
* The RPC client can be found in the `/rpc` directory.
* The HTTP(S) proxy is located in the `/proxy` directory.

### `state` Package

* A new field `State.InitialHeight` has been added to record the initial chain height, which must be `1`
  (not `0`) if starting from height `1`. This can be configured via the genesis field `initial_height`.
* The `state` package now has a `Store` interface. All functions in
  [state/store.go](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/56911ee35298191c95ef1c7d3d5ec508237aaff4/state/store.go#L42-L42)
  are now part of the interface. The interface returns errors on all methods and can be used by calling `state.NewStore(dbm.DB)`.

### `privval` Package

All requests are now accompanied by the chain ID from the network.
This is a optional field and can be ignored by key management systems;
however, if you are using the same key management system for multiple different
blockchains, we recommend that you check the chain ID.


### RPC

* `/unsafe_start_cpu_profiler`, `/unsafe_stop_cpu_profiler` and
  `/unsafe_write_heap_profile` were removed.
   For profiling, please use the pprof server, which can
  be enabled through `--rpc.pprof_laddr=X` flag or `pprof_laddr=X` config setting
  in the rpc section.
* The `Content-Type` header returned on RPC calls is now (correctly) set as `application/json`.

### Version

Version is now set through Go linker flags `ld_flags`. Applications that are using tendermint as a library should set this at compile time.

Example:

```sh
go install -mod=readonly -ldflags "-X github.com/Cryptocurrency-TOOLS/tendermint/version.TMCoreSemVer=$(go list -m github.com/Cryptocurrency-TOOLS/tendermint | sed  's/ /\@/g') -s -w " -trimpath ./cmd
```

Additionally, the exported constant `version.Version` is now `version.TMCoreSemVer`.

## v0.33.4

### Go API

* `rpc/client` HTTP and local clients have been moved into `http` and `local`
  subpackages, and their constructors have been renamed to `New()`.

### Protobuf Changes

When upgrading to version 0.33.4 you will have to fetch the `third_party`
directory along with the updated proto files.

### Block Retention

ResponseCommit added a field for block retention. The application can provide information to Tendermint on how to prune blocks.
If an application would like to not prune any blocks pass a `0` in this field.

```proto
message ResponseCommit {
  // reserve 1
  bytes  data          = 2; // the Merkle root hash
  ++ uint64 retain_height = 3; // the oldest block height to retain ++
}
```

## v0.33.0

This release is not compatible with previous blockchains due to commit becoming
signatures only and fields in the header have been removed.

### Blockchain Protocol

`TotalTxs` and `NumTxs` were removed from the header. `Commit` now consists
mostly of just signatures.

```go
type Commit struct {
	Height     int64
	Round      int
	BlockID    BlockID
	Signatures []CommitSig
}
```

```go
type BlockIDFlag byte

const (
	// BlockIDFlagAbsent - no vote was received from a validator.
	BlockIDFlagAbsent BlockIDFlag = 0x01
	// BlockIDFlagCommit - voted for the Commit.BlockID.
	BlockIDFlagCommit = 0x02
	// BlockIDFlagNil - voted for nil.
	BlockIDFlagNil = 0x03
)

type CommitSig struct {
	BlockIDFlag      BlockIDFlag
	ValidatorAddress Address
	Timestamp        time.Time
	Signature        []byte
}
```

See [\#63](https://github.com/tendermint/spec/pull/63) for the complete spec
change.

### P2P Protocol

The secret connection now includes a transcript hashing. If you want to
implement a handshake (or otherwise have an existing implementation), you'll
need to make the same changes that were made
[here](https://github.com/Cryptocurrency-TOOLS/tendermint/pull/3668).

### Config Changes

You will need to generate a new config if you have used a prior version of tendermint.

Tags have been entirely renamed throughout the codebase to events and there
keys are called
[compositeKeys](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/6d05c531f7efef6f0619155cf10ae8557dd7832f/docs/app-dev/indexing-transactions.md).

Evidence Params has been changed to include duration.

* `consensus_params.evidence.max_age_duration`.
* Renamed `consensus_params.evidence.max_age` to `max_age_num_blocks`.

### Go API

* `libs/common` has been removed in favor of specific pkgs.
    * `async`
    * `service`
    * `rand`
    * `net`
    * `strings`
    * `cmap`
* removal of `errors` pkg

### RPC Changes

* `/validators` is now paginated (default: 30 vals per page)
* `/block_results` response format updated [see RPC docs for details](https://docs.tendermint.com/v0.33/rpc/#/Info/block_results)
* Event suffix has been removed from the ID in event responses
* IDs are now integers not `json-client-XYZ`

## v0.32.0

This release is compatible with previous blockchains,
however the new ABCI Events mechanism may create some complexity
for nodes wishing to continue operation with v0.32 from a previous version.
There are some minor breaking changes to the RPC.

### Config Changes

If you have `db_backend` set to `leveldb` in your config file, please change it
to `goleveldb` or `cleveldb`.

### RPC Changes

The default listen address for the RPC is now `127.0.0.1`. If you want to expose
it publicly, you have to explicitly configure it. Note exposing the RPC to the
public internet may not be safe - endpoints which return a lot of data may
enable resource exhaustion attacks on your node, causing the process to crash.

Any consumers of `/block_results` need to be mindful of the change in all field
names from CamelCase to Snake case, eg. `results.DeliverTx` is now `results.deliver_tx`.
This is a fix, but it's breaking.

### ABCI Changes

ABCI responses which previously had a `Tags` field now have an `Events` field
instead. The original `Tags` field was simply a list of key-value pairs, where
each key effectively represented some attribute of an event occuring in the
blockchain, like `sender`, `receiver`, or `amount`. However, it was difficult to
represent the occurence of multiple events (for instance, multiple transfers) in a single list.
The new `Events` field contains a list of `Event`, where each `Event` is itself a list
of key-value pairs, allowing for more natural expression of multiple events in
eg. a single DeliverTx or EndBlock. Note each `Event` also includes a `Type`, which is meant to categorize the
event.

For transaction indexing, the index key is
prefixed with the event type: `{eventType}.{attributeKey}`.
If the same event type and attribute key appear multiple times, the values are
appended in a list.

To make queries, include the event type as a prefix. For instance if you
previously queried for `recipient = 'XYZ'`, and after the upgrade you name your event `transfer`,
the new query would be for `transfer.recipient = 'XYZ'`.

Note that transactions indexed on a node before upgrading to v0.32 will still be indexed
using the old scheme. For instance, if a node upgraded at height 100,
transactions before 100 would be queried with `recipient = 'XYZ'` and
transactions after 100 would be queried with `transfer.recipient = 'XYZ'`.
While this presents additional complexity to clients, it avoids the need to
reindex. Of course, you can reset the node and sync from scratch to re-index
entirely using the new scheme.

We illustrate further with a more complete example.

Prior to the update, suppose your `ResponseDeliverTx` look like:

```go
abci.ResponseDeliverTx{
  Tags: []kv.Pair{
    {Key: []byte("sender"), Value: []byte("foo")},
    {Key: []byte("recipient"), Value: []byte("bar")},
    {Key: []byte("amount"), Value: []byte("35")},
  }
}
```

The following queries would match this transaction:

```go
query.MustParse("tm.event = 'Tx' AND sender = 'foo'")
query.MustParse("tm.event = 'Tx' AND recipient = 'bar'")
query.MustParse("tm.event = 'Tx' AND sender = 'foo' AND recipient = 'bar'")
```

Following the upgrade, your `ResponseDeliverTx` would look something like:
the following `Events`:

```go
abci.ResponseDeliverTx{
  Events: []abci.Event{
    {
      Type: "transfer",
      Attributes: kv.Pairs{
        {Key: []byte("sender"), Value: []byte("foo")},
        {Key: []byte("recipient"), Value: []byte("bar")},
        {Key: []byte("amount"), Value: []byte("35")},
      },
    }
}
```

Now the following queries would match this transaction:

```go
query.MustParse("tm.event = 'Tx' AND transfer.sender = 'foo'")
query.MustParse("tm.event = 'Tx' AND transfer.recipient = 'bar'")
query.MustParse("tm.event = 'Tx' AND transfer.sender = 'foo' AND transfer.recipient = 'bar'")
```

For further documentation on `Events`, see the [docs](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/60827f75623b92eff132dc0eff5b49d2025c591e/docs/spec/abci/abci.md#events).

### Go Applications

The ABCI Application interface changed slightly so the CheckTx and DeliverTx
methods now take Request structs. The contents of these structs are just the raw
tx bytes, which were previously passed in as the argument.

## v0.31.6

There are no breaking changes in this release except Go API of p2p and
mempool packages. Hovewer, if you're using cleveldb, you'll need to change
the compilation tag:

Use `cleveldb` tag instead of `gcc` to compile Tendermint with CLevelDB or
use `make build_c` / `make install_c` (full instructions can be found at
<https://docs.tendermint.com/v0.33/introduction/install.html#compile-with-cleveldb-support>)

## v0.31.0

This release contains a breaking change to the behavior of the pubsub system.
It also contains some minor breaking changes in the Go API and ABCI.
There are no changes to the block or p2p protocols, so v0.31.0 should work fine
with blockchains created from the v0.30 series.

### RPC

The pubsub no longer blocks on publishing. This may cause some WebSocket (WS) clients to stop working as expected.
If your WS client is not consuming events fast enough, Tendermint can terminate the subscription.
In this case, the WS client will receive an error with description:

```json
{
  "jsonrpc": "2.0",
  "id": "{ID}#event",
  "error": {
    "code": -32000,
    "msg": "Server error",
    "data": "subscription was canceled (reason: client is not pulling messages fast enough)" // or "subscription was canceled (reason: Tendermint exited)"
  }
}

Additionally, there are now limits on the number of subscribers and
subscriptions that can be active at once. See the new
`rpc.max_subscription_clients` and `rpc.max_subscriptions_per_client` values to
configure this.
```

### Applications

Simple rename of `ConsensusParams.BlockSize` to `ConsensusParams.Block`.

The `ConsensusParams.Block.TimeIotaMS` field was also removed. It's configured
in the ConsensusParsm in genesis.

### Go API

See the [CHANGELOG](CHANGELOG.md). These are relatively straight forward.

## v0.30.0

This release contains a breaking change to both the block and p2p protocols,
however it may be compatible with blockchains created with v0.29.0 depending on
the chain history. If your blockchain has not included any pieces of evidence,
or no piece of evidence has been included in more than one block,
and if your application has never returned multiple updates
for the same validator in a single block, then v0.30.0 will work fine with
blockchains created with v0.29.0.

The p2p protocol change is to fix the proposer selection algorithm again.
Note that proposer selection is purely a p2p concern right
now since the algorithm is only relevant during real time consensus.
This change is thus compatible with v0.29.0, but
all nodes must be upgraded to avoid disagreements on the proposer.

### Applications

Applications must ensure they do not return duplicates in
`ResponseEndBlock.ValidatorUpdates`. A pubkey must only appear once per set of
updates. Duplicates will cause irrecoverable failure. If you have a very good
reason why we shouldn't do this, please open an issue.

## v0.29.0

This release contains some breaking changes to the block and p2p protocols,
and will not be compatible with any previous versions of the software, primarily
due to changes in how various data structures are hashed.

Any implementations of Tendermint blockchain verification, including lite clients,
will need to be updated. For specific details:

* [Merkle tree](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/main/spec/blockchain/encoding.md#merkle-trees)
* [ConsensusParams](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/main/spec/blockchain/state.md#consensusparams)

There was also a small change to field ordering in the vote struct. Any
implementations of an out-of-process validator (like a Key-Management Server)
will need to be updated. For specific details:

* [Vote](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/main/spec/consensus/signing.md#votes)

Finally, the proposer selection algorithm continues to evolve. See the
[work-in-progress
specification](https://github.com/Cryptocurrency-TOOLS/tendermint/pull/3140).

For everything else, please see the [CHANGELOG](./CHANGELOG.md#v0.29.0).

## v0.28.0

This release breaks the format for the `priv_validator.json` file
and the protocol used for the external validator process.
It is compatible with v0.27.0 blockchains (neither the BlockProtocol nor the
P2PProtocol have changed).

Please read carefully for details about upgrading.

**Note:** Backup your `config/priv_validator.json`
before proceeding.

### `priv_validator.json`

The `config/priv_validator.json` is now two files:
`config/priv_validator_key.json` and `data/priv_validator_state.json`.
The former contains the key material, the later contains the details on the last
message signed.

When running v0.28.0 for the first time, it will back up any pre-existing
`priv_validator.json` file and proceed to split it into the two new files.
Upgrading should happen automatically without problem.

To upgrade manually, use the provided `privValUpgrade.go` script, with exact paths for the old
`priv_validator.json` and the locations for the two new files. It's recomended
to use the default paths, of `config/priv_validator_key.json` and
`data/priv_validator_state.json`, respectively:

```sh
go run scripts/privValUpgrade.go <old-path> <new-key-path> <new-state-path>
```

### External validator signers

The Unix and TCP implementations of the remote signing validator
have been consolidated into a single implementation.
Thus in both cases, the external process is expected to dial
Tendermint. This is different from how Unix sockets used to work, where
Tendermint dialed the external process.

The `PubKeyMsg` was also split into separate `Request` and `Response` types
for consistency with other messages.

Note that the TCP sockets don't yet use a persistent key,
so while they're encrypted, they can't yet be properly authenticated.
See [#3105](https://github.com/Cryptocurrency-TOOLS/tendermint/issues/3105).
Note the Unix socket has neither encryption nor authentication, but will
add a shared-secret in [#3099](https://github.com/Cryptocurrency-TOOLS/tendermint/issues/3099).

## v0.27.0

This release contains some breaking changes to the block and p2p protocols,
but does not change any core data structures, so it should be compatible with
existing blockchains from the v0.26 series that only used Ed25519 validator keys.
Blockchains using Secp256k1 for validators will not be compatible. This is due
to the fact that we now enforce which key types validators can use as a
consensus param. The default is Ed25519, and Secp256k1 must be activated
explicitly.

It is recommended to upgrade all nodes at once to avoid incompatibilities at the
peer layer - namely, the heartbeat consensus message has been removed (only
relevant if `create_empty_blocks=false` or `create_empty_blocks_interval > 0`),
and the proposer selection algorithm has changed. Since proposer information is
never included in the blockchain, this change only affects the peer layer.

### Go API Changes

#### libs/db

The ReverseIterator API has changed the meaning of `start` and `end`.
Before, iteration was from `start` to `end`, where
`start > end`. Now, iteration is from `end` to `start`, where `start < end`.
The iterator also excludes `end`. This change allows a simplified and more
intuitive logic, aligning the semantic meaning of `start` and `end` in the
`Iterator` and `ReverseIterator`.

### Applications

This release enforces a new consensus parameter, the
ValidatorParams.PubKeyTypes. Applications must ensure that they only return
validator updates with the allowed PubKeyTypes. If a validator update includes a
pubkey type that is not included in the ConsensusParams.Validator.PubKeyTypes,
block execution will fail and the consensus will halt.

By default, only Ed25519 pubkeys may be used for validators. Enabling
Secp256k1 requires explicit modification of the ConsensusParams.
Please update your application accordingly (ie. restrict validators to only be
able to use Ed25519 keys, or explicitly add additional key types to the genesis
file).

## v0.26.0

This release contains a lot of changes to core data types and protocols. It is not
compatible to the old versions and there is no straight forward way to update
old data to be compatible with the new version.

To reset the state do:

```sh
tendermint unsafe_reset_all
```

Here we summarize some other notable changes to be mindful of.

### Config Changes

All timeouts must be changed from integers to strings with their duration, for
instance `flush_throttle_timeout = 100` would be changed to
`flush_throttle_timeout = "100ms"` and `timeout_propose = 3000` would be changed
to `timeout_propose = "3s"`.

### RPC Changes

The default behavior of `/abci_query` has been changed to not return a proof,
and the name of the parameter that controls this has been changed from `trusted`
to `prove`. To get proofs with your queries, ensure you set `prove=true`.

Various version fields like `amino_version`, `p2p_version`, `consensus_version`,
and `rpc_version` have been removed from the `node_info.other` and are
consolidated under the tendermint semantic version (ie. `node_info.version`) and
the new `block` and `p2p` protocol versions under `node_info.protocol_version`.

### ABCI Changes

Field numbers were bumped in the `Header` and `ResponseInfo` messages to make
room for new `version` fields. It should be straight forward to recompile the
protobuf file for these changes.

#### Proofs

The `ResponseQuery.Proof` field is now structured as a `[]ProofOp` to support
generalized Merkle tree constructions where the leaves of one Merkle tree are
the root of another. If you don't need this functionality, and you used to
return `<proof bytes>` here, you should instead return a single `ProofOp` with
just the `Data` field set:

```go
[]ProofOp{
    ProofOp{
        Data: <proof bytes>,
    }
}
```

For more information, see:

* [ADR-026](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/30519e8361c19f4bf320ef4d26288ebc621ad725/docs/architecture/adr-026-general-merkle-proof.md)
* [Relevant ABCI
  documentation](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/30519e8361c19f4bf320ef4d26288ebc621ad725/docs/spec/abci/apps.md#query-proofs)
* [Description of
  keys](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/30519e8361c19f4bf320ef4d26288ebc621ad725/crypto/merkle/proof_key_path.go#L14)

### Go API Changes

#### crypto/merkle

The `merkle.Hasher` interface was removed. Functions which used to take `Hasher`
now simply take `[]byte`. This means that any objects being Merklized should be
serialized before they are passed in.

#### node

The `node.RunForever` function was removed. Signal handling and running forever
should instead be explicitly configured by the caller. See how we do it
[here](https://github.com/Cryptocurrency-TOOLS/tendermint/blob/30519e8361c19f4bf320ef4d26288ebc621ad725/cmd/tendermint/commands/run_node.go#L60).

### Other

All hashes, except for public key addresses, are now 32-bytes.

## v0.25.0

This release has minimal impact.

If you use GasWanted in ABCI and want to enforce it, set the MaxGas in the genesis file (default is no max).

## v0.24.0

New 0.24.0 release contains a lot of changes to the state and types. It's not
compatible to the old versions and there is no straight forward way to update
old data to be compatible with the new version.

To reset the state do:

```sh
tendermint unsafe_reset_all
```

Here we summarize some other notable changes to be mindful of.

### Config changes

`p2p.max_num_peers` was removed in favor of `p2p.max_num_inbound_peers` and
`p2p.max_num_outbound_peers`.

```toml
# Maximum number of inbound peers
max_num_inbound_peers = 40

# Maximum number of outbound peers to connect to, excluding persistent peers
max_num_outbound_peers = 10
```

As you can see, the default ratio of inbound/outbound peers is 4/1. The reason
is we want it to be easier for new nodes to connect to the network. You can
tweak these parameters to alter the network topology.

### RPC Changes

The result of `/commit` used to contain `header` and `commit` fields at the top level. These are now contained under the `signed_header` field.

### ABCI Changes

The header has been upgraded and contains new fields, but none of the existing
fields were changed, except their order.

The `Validator` type was split into two, one containing an `Address` and one
containing a `PubKey`. When processing `RequestBeginBlock`, use the `Validator`
type, which contains just the `Address`. When returning `ResponseEndBlock`, use
the `ValidatorUpdate` type, which contains just the `PubKey`.

### Validator Set Updates

Validator set updates returned in ResponseEndBlock for height `H` used to take
effect immediately at height `H+1`. Now they will be delayed one block, to take
effect at height `H+2`. Note this means that the change will be seen by the ABCI
app in the `RequestBeginBlock.LastCommitInfo` at block `H+3`. Apps were already
required to maintain a map from validator addresses to pubkeys since v0.23 (when
pubkeys were removed from RequestBeginBlock), but now they may need to track
multiple validator sets at once to accomodate this delay.

### Block Size

The `ConsensusParams.BlockSize.MaxTxs` was removed in favour of
`ConsensusParams.BlockSize.MaxBytes`, which is now enforced. This means blocks
are limitted only by byte-size, not by number of transactions.