# Endpoint details

Normative behaviour per method. Encodings, block tags, error codes, address kinds and balance kinds are defined in [README.md](README.md). The one-line overview is [README §4](README.md#4-endpoints).

Every method takes positional parameters unless a method defines a single object parameter. Wrong arity is `-32602`.

## Chain and node identity

All methods in this group take no parameters; any parameter is `-32602`.

### eth_chainId

- **Result** QUANTITY.
- **Behaviour** `0x5EA1ED`.
- **Wallet use** Network detection on connect. Equals the chain id the user added the network with.

### net_version

- **Result** string, decimal.
- **Behaviour** `"6201837"`.
- **Wallet use** Legacy network id requested by older libraries.

### web3_clientVersion

- **Result** string.
- **Behaviour** An implementation name and version, `<client>/<version>`.
- **Wallet use** Client identification in logs and explorers.

### net_listening

- **Result** boolean.
- **Behaviour** `true`. There is no peer set to report on.

### eth_syncing

- **Result** `false`.
- **Behaviour** `false`. The surface exposes no sync state; freshness is observable through `eth_blockNumber`.
- **Wallet use** Wallets consult it before trusting balances.

### eth_accounts

- **Result** address[].
- **Behaviour** `[]`. The surface holds no keys.
- **Wallet use** Wallets that see an empty list never call node-side signing methods, which is why those are [not served](#methods-not-served).

## Fees and gas

The values below are constants. This surface charges no fees.

### eth_gasPrice

- **Parameters** none.
- **Result** QUANTITY.
- **Behaviour** `0x3b9aca00`, 1 gwei.
- **Wallet use** Fee estimate on the Send and confirm screens.

### eth_estimateGas

- **Parameters** 1–2 positional: a transaction object, optional block tag. The transaction object is not inspected.
- **Result** QUANTITY.
- **Behaviour** `0x5208`, 21 000.
- **Wallet use** Gas limit the wallet places in the transaction.

### eth_feeHistory

- **Parameters** 2–3 positional: `blockCount` (QUANTITY), `newestBlock` (block tag), optional `rewardPercentiles` (numbers 0–100).
- **Result** `{ oldestBlock, baseFeePerGas[], gasUsedRatio[], reward[][] }`.
- **Behaviour** Zero-valued arrays sized to the requested range. `oldestBlock` is `newestBlock - blockCount + 1`, truncated at genesis with the arrays shrinking to match, so `oldestBlock + gasUsedRatio.length - 1 = newestBlock`. `baseFeePerGas` has one more entry than `gasUsedRatio`. A tag for `newestBlock` resolves against the indexer head.
- **Errors** `-32602` for `blockCount > 1024`, more than 100 percentiles, a percentile outside 0–100, `blockCount × percentiles > 4096`, or an unrecognised tag.
- **Wallet use** Fee-market probing. Zero base fees together with headers that carry no `baseFeePerGas` select legacy transactions.

### eth_maxPriorityFeePerGas

- **Parameters** none.
- **Result** QUANTITY.
- **Behaviour** `0x0`.

### web3_sha3

- **Parameters** 1 positional: DATA of any even length.
- **Result** DATA, 32 bytes.
- **Behaviour** Keccak-256 of the input bytes. `0x` hashes the empty string.

## Blocks

Headers are synthesized from the indexer's block query into the pre-London header shape. Fields Midnight has no analogue for carry fixed values rather than being omitted:

| Field | Value |
|---|---|
| `nonce` | `0x0000000000000000` |
| `sha3Uncles` | `0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347` |
| `logsBloom` | 256 zero bytes |
| `transactionsRoot`, `stateRoot`, `receiptsRoot`, `mixHash` | zero hash |
| `difficulty`, `totalDifficulty`, `gasUsed`, `size` | `0x0` |
| `extraData` | `0x` |
| `gasLimit` | `0x1c9c380` |
| `uncles` | `[]` |
| `miner` | the block author when it is 20 bytes, else the zero address |
| `parentHash` | the parent's hash; the zero hash at height 0 |
| `timestamp` | UNIX seconds |
| `baseFeePerGas` | absent |

### eth_blockNumber

- **Parameters** none.
- **Result** QUANTITY.
- **Behaviour** The indexer head height; `0x0` when the indexer reports no block.
- **Wallet use** Block polling and confirmation counting.

### eth_getBlockByNumber

- **Parameters** 2 positional: block tag, `fullTransactions` boolean (required).
- **Result** Block or `null`.
- **Behaviour** `null` for a height not on chain. With `false` the `transactions` array holds hashes; with `true` it holds transaction objects built by the [transaction synthesis](#transactions-and-receipts).
- **Errors** `-32602` for a non-boolean flag or an invalid tag.
- **Wallet use** Wallets read `latest` to decide EIP-1559 support; the missing `baseFeePerGas` selects legacy transactions.

### eth_getBlockByHash

- **Parameters** 2 positional: 32-byte block hash, `fullTransactions` boolean.
- **Result** Block or `null`.
- **Behaviour** Same synthesis, addressed by hash. Unknown hash is `null`.

### eth_getBlockTransactionCountByHash

- **Parameters** 1 positional: 32-byte block hash.
- **Result** QUANTITY or `null`.
- **Behaviour** Length of the block's transaction list; unknown block is `null`.

### eth_getBlockTransactionCountByNumber

- **Parameters** 1 positional: block tag.
- **Result** QUANTITY or `null`.
- **Behaviour** Same, by tag.

### eth_getBlockReceipts

- **Parameters** 1 positional: a block tag, a 32-byte block hash, or the object forms `{ blockNumber }` and `{ blockHash, requireCanonical }`.
- **Result** Receipt[] or `null`.
- **Behaviour** One receipt per transaction the block lists, in block order, in the [receipt shape](#eth_gettransactionreceipt). Unknown block is `null`; an empty block is `[]`. The array stays index-aligned with the block's transaction list: a transaction the block lists but the transaction lookup cannot resolve yields a placeholder receipt with `status 0x0` and `gasUsed 0x0` rather than an omission.
- **Wallet use** Explorers and indexers batch-fetching receipts.

## Accounts and tokens

Every method here accepts an optional block tag as its last parameter, validates it, and answers current state. The surface keeps no historical native balances.

### eth_getBalance

- **Parameters** 1–2 positional: 20-byte address, optional block tag.
- **Result** QUANTITY.
- **Behaviour** The address's unshielded NIGHT in STAR × 10^12. For kinds `midnight` and `ethereum` the value is the sum of unspent NIGHT UTXOs of the bound identity in the UTXO balance store. For kind `contract` it is the NIGHT entry of the contract's latest action's unshielded balances. An address without a registry entry, or of a token or protocol kind, answers `0x0`.
- **Wallet use** The account headline in the wallet.

### eth_getTransactionCount

- **Parameters** 1–2 positional: address, optional block tag.
- **Result** QUANTITY.
- **Behaviour** The number of indexed transactions sent by the address. Monotonic over the indexed set; not a state-trie nonce. Unknown address is `0x0`.
- **Wallet use** Nonce for the wallet's next transaction.

### eth_getCode

- **Parameters** 1–2 positional: address, optional block tag.
- **Result** DATA.
- **Behaviour** `0x60006000`, a non-executable marker, for every registry kind other than `midnight` and `ethereum`; `0x` otherwise. Contract, token and protocol addresses therefore all read as code-bearing.
- **Wallet use** Wallets treat empty code as a plain account and refuse token flows against it.

### eth_call

- **Parameters** 1–2 positional: a call object `{ to, data, … }`, optional block tag. `from`, `gas`, `gasPrice` and `value` are accepted and ignored.
- **Result** DATA, ABI-encoded.
- **Behaviour** There is no contract execution. The surface dispatches on the registry kind of `to` and on the 4-byte selector in `data`. Five ERC-20 view selectors are answered:

  | Selector | Function | Return encoding |
  |---|---|---|
  | `70a08231` | `balanceOf(address)` | one `uint256` word |
  | `18160ddd` | `totalSupply()` | one `uint256` word |
  | `313ce567` | `decimals()` | one `uint256` word |
  | `95d89b41` | `symbol()` | ABI `string`: offset word, length word, padded bytes |
  | `06fdde03` | `name()` | ABI `string` |

  By kind of `to`:

  | Kind | `balanceOf` | `totalSupply` | `decimals`, `symbol`, `name` |
  |---|---|---|---|
  | `contract` | The holder's balance in the contract's ledger state, obtained by running the contract's read circuit locally against the state from `contractAction(address).state`, or by decoding the ledger field with the compiled module's `ledger(state)` accessor. The holder argument is translated per [README §2](README.md#2-addresses). When a block tag names a height within the indexer's retention window, the state at that block is used. | From the ledger state | Token manifest |
  | `token-unshielded` | Sum of the holder's unspent UTXOs of that color in the UTXO balance store | Store-wide sum for the color | Token manifest |
  | `token-shielded` | `0x` on the shared surface; a real value only on a per-user session surface (README §3) | `0x` | Token manifest |
  | `protocol` (DUST) | The account's DUST at the time of the call: generated capacity minus reported spends (README §3) | `0x` | `decimals` = 15; `symbol` = `DUST` |

  Any other target, selector, or calldata shorter than four bytes returns `0x`.
- **Errors** `-32602` for wrong arity, a non-object call, or a malformed address. A well-formed call the surface cannot execute is `0x`, never an error.
- **Wallet use** Token import: wallets call `symbol()`, `decimals()` and `balanceOf(you)`. Token display. The Send flow that produces `transfer(address,uint256)` calldata. Pasting a token address into a wallet's custom-token form auto-fills the rest from these answers.

## Transactions and receipts

Transactions are synthesized into the Ethereum shape from what Midnight records. Constant fields:

| Field | Value |
|---|---|
| `gas` | `0x5208` |
| `gasPrice` | `0x3b9aca00` |
| `input` | `0x` |
| `value` | `0x0` — the transferred amount is in the Transfer log, not the transaction |
| `type` | `0x0` |
| `v`, `r`, `s` | `0x0`, zero hash, zero hash |

A transaction that entered through `eth_sendRawTransaction` has two identities: the hash the relayer returned, which the wallet polls with, and the Midnight hash the chain knows it by. The relayer records the pair in the transaction index; that mapping is the only thing this specification requires of it. Every method echoes the identifier it was queried by and positions the transaction by the Midnight hash.

### eth_getTransactionByHash

- **Parameters** 1 positional: 32-byte transaction hash.
- **Result** Transaction or `null`.
- **Behaviour** Transaction index first, indexer second. `from` and `to` are the mapped 20-byte addresses when the index has them, else the zero address; `nonce` is the count of the sender's earlier indexed transactions. `blockHash`, `blockNumber` and `transactionIndex` come from the block that contains the Midnight hash. A hash neither source can place is `null`, never an error.
- **Wallet use** Wallets poll this with the hash `eth_sendRawTransaction` returned.

### eth_getTransactionReceipt

- **Parameters** 1 positional: 32-byte transaction hash.
- **Result** Receipt or `null`.
- **Behaviour** One receipt shape, shared with `eth_getBlockReceipts`:

  | Field | Value |
  |---|---|
  | `status` | `0x1` only when the Midnight result is SUCCESS and every segment succeeded, else `0x0` |
  | `gasUsed`, `cumulativeGasUsed` | the recorded fee |
  | `contractAddress` | `null` |
  | `type` | `0x0` |
  | `effectiveGasPrice` | `0x3b9aca00` |
  | `logs` | the transaction's rows from the log store, identical objects to `eth_getLogs`, joined on the Midnight hash |
  | `logsBloom` | computed from `logs` |

  For a transaction that entered through `eth_sendRawTransaction`, `transactionHash` is the hash the relayer returned and `logs[].transactionHash` the Midnight hash. This difference is by design.
- **Wallet use** Confirmation and token-transfer detection.

### eth_getTransactionByBlockHashAndIndex

- **Parameters** 2 positional: 32-byte block hash, index (QUANTITY).
- **Result** Transaction or `null`.
- **Behaviour** The transaction at that position in the block's list; the transaction index enriches `from`, `to` and `nonce`. Unknown block or out-of-range index is `null`.

### eth_getTransactionByBlockNumberAndIndex

- **Parameters** 2 positional: block tag, index (QUANTITY).
- **Result** Transaction or `null`.
- **Behaviour** Same, by tag.

## Logs and subscriptions

Logs are derived from the indexer's contract events and held in the log store; no log read depends on the indexer being reachable.

Mapping rules:

- An unshielded Spend paired with an unshielded Receive in the same transaction, with equal domain separator, token type and amount, becomes one `Transfer(address indexed from, address indexed to, uint256 value)`. Pairing is FIFO by event id within the transaction.
- An unpaired Spend is a burn: `Transfer(from = sender, to = 0x0)`. An unpaired Receive is a mint: `Transfer(from = 0x0, to = recipient)`.
- The log's `address` is the asset's EVM address per README §2: the contract for ledger-managed balances, the color address for UTXO-based tokens.
- `topic1` and `topic2` are the 20-byte addresses left-padded to 32 bytes. For ERC-721 profiles `topic3` is the token id and `data` is empty; for ERC-20 profiles `data` is the amount as a `uint256` word.
- Every other event type keeps a lossless topic `keccak256("Midnight<TypeName>()")` with its fields as 32-byte words in `data`.
- `logIndex` is the 0-based position within the transaction; `removed` is always `false`.

### eth_getLogs

- **Parameters** 1 positional: a filter object.

  | Field | Accepted | Meaning |
  |---|---|---|
  | `address` | absent, one address, or an array | OR over addresses; an address nothing emitted from yields `[]` |
  | `topics` | positional array, at most four entries, each `null`, a topic, or a nested array | `null` is a wildcard over the value; a nested array is OR at that position; a filter of length L matches only logs with at least L topics |
  | `fromBlock`, `toBlock` | QUANTITY or block tag, both default `latest` | an inverted range yields `[]` |
  | `blockHash` | 32 bytes | mutually exclusive with the range |

- **Result** Log[] — `{ address, topics[], data, blockNumber, blockHash, transactionHash, transactionIndex, logIndex, removed }`, ordered by block number, transaction index, log index.
- **Errors** `-32602` for a missing or non-object filter, `blockHash` combined with a range, a malformed address or topic, or more than four topic positions. More than 10 000 matching rows is `-32005`.
- **Wallet use** Token activity and dapp event reads. Libraries poll this in place of installed filters.

### eth_subscribe

WebSocket surface only.

- **Parameters** 1–2 positional: the kind, plus a filter object for `logs`.
- **Result** QUANTITY subscription id, unique across connections.
- **Behaviour** Kinds: `logs`, delivering each committed log as one notification in the `eth_getLogs` object shape, filtered with the same rules; `newHeads`, delivering `{ number, hash, parentHash, timestamp }` for every new indexer head. Notifications are `{ "jsonrpc": "2.0", "method": "eth_subscription", "params": { subscription, result } }`. A logs subscriber is never notified of a log that a rolled-back write then removed.
- **Errors** `-32602` for any other kind, including `newPendingTransactions` and `syncing`, or a malformed filter.

### eth_unsubscribe

WebSocket surface only.

- **Parameters** 1 positional: a subscription id.
- **Result** boolean.
- **Behaviour** `true` when a subscription was removed, `false` for any other id. Subscriptions end with their connection.

## Write path and discovery

### eth_sendRawTransaction

- **Parameters** 1 positional: the payload as DATA.
- **Result** DATA, 32 bytes.
- **Behaviour** The payload is forwarded to the configured relayer unchanged and the relayer's 32-byte hash is returned. Relayer error codes and messages pass through verbatim. The payload format, its validation and what the relayer does with it are outside this specification; the relayer's only obligation to this surface is to record the returned hash against the resulting Midnight transaction in the transaction index. When no relayer is configured the method answers `-32004` with reason `write path not configured`.
- **Errors** `-32602` for a missing or non-string parameter.
- **Wallet use** The wallet's Send. Success means the relayer accepted the payload; the wallet observes the outcome through `eth_getTransactionByHash` and `eth_getTransactionReceipt` with the returned hash.

### midnight_getTokenBalances

- **Parameters** 1–3 positional: `address`; optional `tokenSpec`, either `"erc20"` for every registered token or an array of token addresses; optional `{ pageKey, maxCount }` with `maxCount ≤ 100`.
- **Result** `{ address, tokenBalances: [{ contractAddress, tokenBalance, kind, symbol, decimals }], pageKey? }`.
- **Behaviour** Every token with a non-zero balance for the address: ledger-managed tokens, unshielded colors and DUST, each with its registry `kind`. `tokenBalance` is the same value `eth_call balanceOf` answers, as a QUANTITY; `symbol` and `decimals` come from the token manifest. Shielded colors are omitted on the shared surface. The shape follows the `alchemy_getTokenBalances` convention so tooling written against it needs only a method rename.
- **Wallet use** Discovery for a companion page that then calls `wallet_watchAsset` once per token, and for explorer holdings views.

### rpc.discover

- **Parameters** none.
- **Result** OpenRPC document.
- **Behaviour** Describes every method in this specification, including the `midnight_` namespace, per EIP-1901.

## Methods not served

Spec-defined methods this surface knows but does not serve answer `-32004 Method not supported` with `data: { method, classification, reason, documentation }`, so a client can fall back instead of concluding the endpoint is broken. Genuinely unknown names answer `-32601`.

| Classification | Meaning | Methods |
|---|---|---|
| `backlog` | Implementable, deferred. Clients poll `eth_getLogs` and use `eth_subscribe` for live tails; there is no mempool for pending filters. | `eth_newFilter`, `eth_newBlockFilter`, `eth_newPendingTransactionFilter`, `eth_getFilterChanges`, `eth_getFilterLogs`, `eth_uninstallFilter`, `eth_capabilities`, `eth_config` |
| `intentionally-absent` | The capability lives elsewhere: signing is the wallet's, and there is no mining identity. | `eth_sendTransaction`, `eth_sign`, `eth_signTransaction`, `eth_coinbase` |
| `n/a-by-design` | Structurally impossible: no EVM execution engine, storage trie or state proofs; contract state is a ledger blob. | `eth_getStorageAt`, `eth_getStorageValues`, `eth_getProof`, `eth_createAccessList`, `eth_getBlockAccessList`, `eth_simulateV1`, `eth_fillTransaction`, `eth_baseFee`, `eth_blobBaseFee` |

The `debug_`, `trace_`, `txpool_`, `personal_`, `admin_` and mining namespaces, and unknown `midnight_` names, answer `-32601`.

## Transport and envelope

Applies to both surfaces.

| Aspect | Behaviour |
|---|---|
| Envelope | `jsonrpc: "2.0"` and a string `method` are required, else `-32600`. Unknown members are ignored. `id` is echoed for string, number and null; other types are `-32600` with `id: null`. |
| Notifications | A request without `id` receives no response. An HTTP payload consisting only of notifications answers 204 with an empty body. |
| Params | An array or an object if present; other types are `-32602`. |
| Batches | Arrays are dispatched with bounded concurrency and answered in request order. One failing entry never fails the batch. An empty array is a single `-32600`. More than 100 entries is `-32600`. Responses beyond 1 MiB answer `-32005` for the remaining entries. |
| Limits | Request bodies over 1 MiB are refused with HTTP 413 and `-32600`. Handler results are canonicalized through JSON before serialization, so BigInt, cycles and `toJSON` side effects never reach the wire. |
| HTTP | `POST` only; other verbs answer 405 with `Allow: POST, OPTIONS`. `OPTIONS` answers 204 with CORS headers. CORS origins are configurable; the default for a local deployment is allow-all and a public deployment sets an allowlist. |
| WebSocket | RFC 6455 text frames. The same envelope rules as HTTP, plus subscription notifications. A connection's subscriptions end when it closes. |

## Handled inside the wallet

Methods a dapp sends to the wallet extension. They never reach this surface, but they shape what it has to answer.

| Method | Result | Behaviour | Consequence for this surface |
|---|---|---|---|
| `eth_requestAccounts` | address[] | The wallet returns the user's selected account after the connect prompt. | None. |
| `wallet_addEthereumChain`, `wallet_switchEthereumChain` | null | The wallet stores chain 6201837 with the RPC URL and `nativeCurrency { name: NIGHT, symbol: NIGHT, decimals: 18 }`, then switches to it. | Wallets require 18 decimals for the native currency, which fixes the 10^12 scale. A companion page switches chains before adding tokens. |
| `wallet_watchAsset` (EIP-747) | boolean | The wallet prompts once per asset, `{ type: "ERC20", options: { address, symbol, decimals, image } }` or ERC721/ERC1155 with `tokenId`, and remembers it for that account and network. | The only path that adds tokens without manual entry on a custom chain. Wallets verify `symbol()` and `decimals()` through `eth_call` and reject a mismatch, which is why the token manifest is the single metadata source. Symbols are at most 5 characters. |
| `personal_sign`, `eth_signTypedData_v4` | DATA signature | The wallet signs with the user's key. | Signed payloads are consumed by the relayer; outside this specification. |

## Conformance checks

An implementation conforms when every statement below holds against a running stack.

1. A wallet adds the network with chain id 6201837, native currency NIGHT at 18 decimals, and connects without a currency-symbol or chain-id warning; `eth_chainId` answers `0x5EA1ED` and `net_version` `"6201837"`.
2. For a registered identity holding *n* STAR of NIGHT, `eth_getBalance` answers *n* × 10^12 as a QUANTITY; for an unregistered address it answers `0x0`.
3. Pasting any registered token address into the wallet's custom-token form fills symbol and decimals from `eth_call`, and the shown balance equals the store or ledger-state balance for that kind.
4. The DUST address answers `symbol()` = `DUST`, `decimals()` = 15, and a `balanceOf` that changes between two calls a minute apart for a generating account.
5. A color minted both shielded and unshielded appears as two token addresses; the unshielded one answers a balance, the shielded one answers `0x` for `balanceOf` on the shared surface.
6. A contract minting two colors yields Transfer logs under two distinct `address` values, neither equal to the contract's own address, and `midnight_getTokenBalances` lists them separately.
7. A payload the configured relayer accepts through `eth_sendRawTransaction` becomes visible through `eth_getTransactionByHash` and `eth_getTransactionReceipt` under the hash the relayer returned, with a `logsBloom` that matches the receipt's `logs`.
8. Every spec-defined method not listed as served answers `-32004` with a `data.classification`; an invented name answers `-32601`; `rpc.discover` lists exactly the served methods.
9. An `eth_getLogs` filter matching more than 10 000 rows answers `-32005`; a filter over an empty range answers `[]`.
10. A library's WebSocket provider connects to port 10021, reads `eth_chainId` on that connection, and receives a `newHeads` notification for the next indexed block.
