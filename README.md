# Midnight EVM JSON-RPC Specification

**Status:** Draft · 2026-09-08

A JSON-RPC 2.0 surface that lets Ethereum wallets and tooling read Midnight. Writes are not interpreted here: `eth_sendRawTransaction` forwards its payload unchanged to a configured relayer whose behaviour is outside this specification. This repository is a specification, not an implementation. Any system that serves the methods listed in [Endpoints](#4-endpoints) with the behaviour defined in [ENDPOINTS-DETAILS.md](ENDPOINTS-DETAILS.md) conforms.

The surface is not an Ethereum node. There is no EVM execution, no storage trie and no mempool. Every result is derived from the Midnight indexer, from data the implementation derives from the indexer, from configuration, or from a relayer. Behaviour not described here is out of scope.

## Reading order

| File | Contents |
|---|---|
| [README.md](README.md) | Key concepts: conventions, addresses, balance kinds; one row per endpoint with its result type and the Midnight data behind it |
| [ENDPOINTS-DETAILS.md](ENDPOINTS-DETAILS.md) | Full behaviour per endpoint, methods not served, transport rules, conformance checks |

## 1. Conventions

| Term | Definition |
|---|---|
| **Chain id** | `6201837` (`0x5EA1ED`). Reported by `eth_chainId` as hex and by `net_version` as a decimal string. |
| **Native currency** | NIGHT, presented with 18 decimals: STAR amounts are padded by 10^12. |
| **QUANTITY** | JSON-RPC data type for an integer: `0x`-prefixed hex, no leading zeros, `0x0` for zero. Used for block numbers, balances, gas, nonces, indexes and subscription ids. Non-canonical input is `-32602`. |
| **DATA** | JSON-RPC data type for a byte string: `0x`-prefixed hex, two characters per byte, leading zeros kept, lowercase on output. Length is fixed by meaning: addresses 20 bytes, hashes and topics 32 bytes. Wrong length is `-32602`. |
| **Block tag** | The surface returns the best available candidate. `latest`, `pending`, `safe` and `finalized` all resolve to the tip; `earliest` is height 0; a QUANTITY is a height, at most 2^31 - 1. |
| **Midnight hex** | Midnight values are unprefixed hex; this surface adds the `0x` prefix. |
| **Surfaces** | HTTP on port 8545 and WebSocket on port 10021. Both serve every method; WebSocket additionally serves `eth_subscribe` and `eth_unsubscribe`. Both apply the same [envelope rules](ENDPOINTS-DETAILS.md#transport-and-envelope). |
| **Error codes** | `-32700` parse · `-32600` invalid request · `-32601` unknown method · `-32602` invalid params · `-32603` internal · `-32004` method known and not served, with `data` · `-32005` limit exceeded. Deliberate errors carry one of these codes; anything else is sanitized to `-32603`. |
| **Derived data** | Four datasets an implementation builds from the indexer and the relayer, held however it chooses: the **registry** of EVM addresses ([Addresses](#2-addresses)); **unshielded balances**, unspent value per identity and token type, folded from `unshieldedTransactions`; the **transaction index**, transactions by hash with block position, result, fee, mapped sender and recipient, and the relayer's returned hash; and **derived logs**, EVM-shaped logs mapped from `contractEvents`. |
| **Token manifest** | Configuration listing every token the surface serves: address kind, symbol, name, decimals and, for contract tokens, the compiled contract module. It is the single metadata source for `eth_call`, `midnight_getTokenBalances` and derived logs. |

## 2. Addresses

Midnight identities are 32 bytes; EVM addresses are 20. Every EVM address the surface answers for is derived by one of five rules and recorded in the registry with its kind. The registry is the discriminator: every read dispatches on the kind of the address it was asked about, never on the shape of the address.

| Kind | Derivation · registry kind | Rules |
|---|---|---|
| **Midnight user identity** | `keccak256(identity)[12:32]` · `midnight` | Registered at first sighting. `eth_getCode` is `0x`. |
| **Ethereum-native identity** | The 20 bytes verbatim; on the Midnight side, the address zero-left-padded to 32 bytes · `ethereum` | A 32-byte identity with 12 leading zero bytes is read as this kind. Midnight-side balances attach only through a binding to a Midnight identity; how bindings are established is out of scope. |
| **Compact contract** | `keccak256(contractAddress)[12:32]` · `contract` | The ERC-20 address for balances the contract manages in its ledger, and the holder address for value the contract owns. |
| **Token color** | `keccak256("midnight-evm:token:" ‖ tag ‖ raw)[12:32]`, `tag` = `unshielded` or `shielded` · `token-unshielded`, `token-shielded` | The raw color is identical in both pools; the tag makes two addresses. A contract minting two colors yields two ERC-20s, neither the contract's own address. |
| **Protocol asset** | Fixed constant; DUST is `0x1111111111111111111111111111111111111111` · `protocol` | DUST has no raw color. Further protocol assets take further constants. |

**Registry rules.** The registry is unique on EVM address. Two identities deriving the same address is a hard error that aborts the write; it is never a merge. Every kind other than `midnight` and `ethereum` answers the code marker in `eth_getCode`. Token metadata comes from the token manifest.

**How entries arise.** Contract and protocol entries come from the token manifest and the fixed constants. Token colors are registered from the manifest or at first sighting of a new token type in UTXOs or contract events. Midnight identities are registered at first sighting as UTXO owners, as senders or recipients in contract events, or as holders in decoded contract state. Ethereum-native identities are registered when a 32-byte identity with 12 leading zero bytes is sighted, or when a binding is established. Which Midnight addresses an implementation observes is its own choice: a configured list followed through per-address `unshieldedTransactions` subscriptions, or every block. An address with no entry is not an error: balance reads answer `0x0` and `eth_getCode` answers `0x`.

**Argument translation.** An ERC-20 argument arrives as a 20-byte address and is translated to the Compact type through the registry: an `ethereum` kind becomes the address zero-left-padded in the user branch, a `midnight` kind becomes its stored 32-byte identity, a `contract` kind goes into the contract branch. Calldata is never forwarded into a circuit unchanged.

## 3. Balance kinds

Six kinds of value exist on Midnight and they are read from different places. The wallet-facing method is `eth_getBalance` for NIGHT and `eth_call` for everything else. Where a row says **needs keys**, the number cannot be computed without the holder's secret material and the shared surface does not compute it.

| Balance | Where it lives | Source | Exposure | Requires |
|---|---|---|---|---|
| **NIGHT** (unshielded, zero color) | Unshielded UTXO ledger; owner is an unshielded address. | `unshieldedTransactions(address)`, folded into unshielded balances. A contract's NIGHT: the NIGHT entry of `contractAction(address).unshieldedBalances`. | `eth_getBalance`, × 10^12. | A registry binding from the wallet's address to the Midnight identity. |
| **DUST** (tag dust, no color) · needs keys | Dust ledger; generated by NIGHT UTXOs registered for dust generation, decaying toward a cap. SPECK, 15 decimals. | Generation per dust address from the `dustGenerations(dustAddress, …)` subscription and `dustLedgerEvents`. Spends are keyed by nullifier via `dustNullifierTransactions(prefixes)`, which the holder derives from the dust secret key. Balance at *t* = generated capacity at *t* minus spends. | Virtual ERC-20 at `0x1111…1111`. `balanceOf` is generated capacity at call time minus reported spends; without reported spends it is an upper bound and the token name says so. Non-transferable. | A binding to a dust address; the dust secret key client-side or client-reported nullifiers for an exact figure. |
| **Native unshielded tokens** (colors, tag unshielded) | Unshielded UTXO ledger; color = `rawTokenType(domainSep, contract)`. | The same `unshieldedTransactions` stream, folded per identity and token type. Contract holdings from `unshieldedBalances`. Activity from `Transaction.unshieldedCreatedOutputs` / `unshieldedSpentOutputs` and paired Spend/Receive contract events. | One virtual ERC-20 per color at its `token-unshielded` address; `balanceOf` from unshielded balances, `totalSupply` the sum over all holders; Transfer logs carry the color address. | The same registry binding as NIGHT. |
| **Native shielded tokens** (colors, tag shielded) · needs keys | Zswap pool; same raw color as the unshielded variant; spends by nullifier. | `zswapLedgerEvents(id)` replayed client-side with the Zswap secret keys, or `connect(viewingKey)` + `shieldedTransactions(sessionId)` for received outputs only; spends need the coin secret key. | Shared surface answers metadata only. Balances are served client-side by a wallet extension holding the keys, or by a per-user session surface bound to a viewing key and fed client-reported nullifiers, at the `token-shielded` address. | Zswap secret keys client-side, or a viewing key plus reported nullifiers on a per-user surface. |
| **Ledger unshielded** (contract account model) | The contract's public ledger fields, e.g. `Map<holder, Uint<128>>`; Ethereum-native holders as the 20-byte address zero-padded. | `contract(address).state` or `contractAction(address, offset).state`, read by executing the contract's read circuit locally against that state or by decoding the ledger field with the compiled module's `ledger(state)` accessor. Transfer logs from `contractEvents` give the same balances by fold and serve activity. | `eth_call` ERC-20 views at the contract's address; Transfer logs through `eth_getLogs`. | The contract's compiled module in the token manifest. |
| **Ledger shielded** (contract account model, private) · needs keys | (a) Shielded coins the contract holds, in its ledger as `QualifiedShieldedCoinInfo` and in `contractAction.zswapState`. (b) Holder balances kept as commitments; values live in the holder's private state. | (a) The same state read. (b) Only the holder, through the dapp's private state or a wallet extension; a contract may disclose through its own events. | (a) `balanceOf(contractAddress)` on the color's shielded ERC-20 view. (b) Client-side only; the shared surface answers `0x`. | (a) The compiled module. (b) The holder's private state. |

## 4. Endpoints

One row per method. **Result** is the JSON-RPC result type, using the encodings in [Conventions](#1-conventions). **Midnight data** names what the value is built from, in the vocabulary of [Conventions](#1-conventions), [Addresses](#2-addresses) and [Balance kinds](#3-balance-kinds), without the reasoning. Each method links to its section in [ENDPOINTS-DETAILS.md](ENDPOINTS-DETAILS.md), which defines parameters, behaviour and errors, plus the [transport and envelope rules](ENDPOINTS-DETAILS.md#transport-and-envelope) and the [conformance checks](ENDPOINTS-DETAILS.md#conformance-checks).

29 methods are served on both HTTP and WebSocket, plus 2 subscription methods on WebSocket only. 21 further spec-defined methods answer `-32004` ([methods not served](ENDPOINTS-DETAILS.md#methods-not-served)). Any other name answers `-32601`.

### Chain and node identity

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_chainId`](ENDPOINTS-DETAILS.md#eth_chainid) | QUANTITY | Constant `0x5EA1ED` |
| [`net_version`](ENDPOINTS-DETAILS.md#net_version) | string | Constant `"6201837"` |
| [`web3_clientVersion`](ENDPOINTS-DETAILS.md#web3_clientversion) | string | Implementation name and version |
| [`net_listening`](ENDPOINTS-DETAILS.md#net_listening) | boolean | Constant `true` |
| [`eth_syncing`](ENDPOINTS-DETAILS.md#eth_syncing) | `false` | Constant `false` |
| [`eth_accounts`](ENDPOINTS-DETAILS.md#eth_accounts) | address[] | Constant `[]` |

### Fees and gas

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_gasPrice`](ENDPOINTS-DETAILS.md#eth_gasprice) | QUANTITY | Constant `0x3b9aca00` |
| [`eth_estimateGas`](ENDPOINTS-DETAILS.md#eth_estimategas) | QUANTITY | Constant `0x5208` |
| [`eth_feeHistory`](ENDPOINTS-DETAILS.md#eth_feehistory) | `{ oldestBlock, baseFeePerGas[], gasUsedRatio[], reward[][] }` | Zero arrays sized from the requested range; indexer head height when `newestBlock` is a tag |
| [`eth_maxPriorityFeePerGas`](ENDPOINTS-DETAILS.md#eth_maxpriorityfeepergas) | QUANTITY | Constant `0x0` |
| [`web3_sha3`](ENDPOINTS-DETAILS.md#web3_sha3) | DATA, 32 bytes | Keccak-256 of the input |

### Blocks

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_blockNumber`](ENDPOINTS-DETAILS.md#eth_blocknumber) | QUANTITY | Indexer head height |
| [`eth_getBlockByNumber`](ENDPOINTS-DETAILS.md#eth_getblockbynumber) | Block \| null | Indexer block at the tag: height, hash, parent hash, author, timestamp, transaction hashes; fixed values for every other header field |
| [`eth_getBlockByHash`](ENDPOINTS-DETAILS.md#eth_getblockbyhash) | Block \| null | Indexer block by hash, same fields |
| [`eth_getBlockTransactionCountByHash`](ENDPOINTS-DETAILS.md#eth_getblocktransactioncountbyhash) | QUANTITY \| null | Length of the indexer block's transaction list |
| [`eth_getBlockTransactionCountByNumber`](ENDPOINTS-DETAILS.md#eth_getblocktransactioncountbynumber) | QUANTITY \| null | Same, by tag |
| [`eth_getBlockReceipts`](ENDPOINTS-DETAILS.md#eth_getblockreceipts) | Receipt[] \| null | Indexer block's transaction list; per transaction, the data of `eth_getTransactionReceipt` |

### Accounts and tokens

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_getBalance`](ENDPOINTS-DETAILS.md#eth_getbalance) | QUANTITY | Sum of unspent NIGHT UTXOs of the bound identity from unshielded balances, × 10^12; for a `contract` kind, the NIGHT entry of `contractAction.unshieldedBalances` |
| [`eth_getTransactionCount`](ENDPOINTS-DETAILS.md#eth_gettransactioncount) | QUANTITY | Count of transaction-index rows sent by the address |
| [`eth_getCode`](ENDPOINTS-DETAILS.md#eth_getcode) | DATA | Registry kind of the address |
| [`eth_call`](ENDPOINTS-DETAILS.md#eth_call) | DATA, ABI-encoded | By registry kind of `to`: `contract` → contract ledger state from `contractAction.state`; `token-unshielded` → unshielded balances; `token-shielded` → metadata only; `protocol` DUST → dust generation data at call time. Symbol, name and decimals from the token manifest |

### Transactions and receipts

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_getTransactionByHash`](ENDPOINTS-DETAILS.md#eth_gettransactionbyhash) | Transaction \| null | Transaction-index row: sender, recipient, sent-count nonce; indexer block containing the Midnight hash for position |
| [`eth_getTransactionReceipt`](ENDPOINTS-DETAILS.md#eth_gettransactionreceipt) | Receipt \| null | Indexer transaction result, segment results and fee; derived logs keyed by the Midnight hash |
| [`eth_getTransactionByBlockHashAndIndex`](ENDPOINTS-DETAILS.md#eth_gettransactionbyblockhashandindex) | Transaction \| null | Indexer block transaction list at the index; transaction-index row for sender, recipient and nonce |
| [`eth_getTransactionByBlockNumberAndIndex`](ENDPOINTS-DETAILS.md#eth_gettransactionbyblocknumberandindex) | Transaction \| null | Same, by tag |

### Logs and subscriptions

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_getLogs`](ENDPOINTS-DETAILS.md#eth_getlogs) | Log[] | Derived logs: paired unshielded Spend/Receive contract events as `Transfer`, unpaired ones as mint or burn, every other event type under its `Midnight<Type>()` topic |
| [`eth_subscribe`](ENDPOINTS-DETAILS.md#eth_subscribe) (WebSocket) | QUANTITY subscription id | `logs`: derived logs as they commit; `newHeads`: indexer head |
| [`eth_unsubscribe`](ENDPOINTS-DETAILS.md#eth_unsubscribe) (WebSocket) | boolean | Subscription table |

### Write path and discovery

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_sendRawTransaction`](ENDPOINTS-DETAILS.md#eth_sendrawtransaction) | DATA, 32 bytes | The 32-byte hash the configured relayer returns for the forwarded payload |
| [`midnight_getTokenBalances`](ENDPOINTS-DETAILS.md#midnight_gettokenbalances) | `{ address, tokenBalances[], pageKey? }` | Unshielded balances, contract ledger state, dust generation data; metadata from the token manifest |
| [`rpc.discover`](ENDPOINTS-DETAILS.md#rpcdiscover) | OpenRPC document | Configuration |

### Methods not served

Each answers `-32004` with `data: { method, classification, reason, documentation }`. See [methods not served](ENDPOINTS-DETAILS.md#methods-not-served).

| Methods | Result | Classification |
|---|---|---|
| `eth_newFilter`, `eth_newBlockFilter`, `eth_newPendingTransactionFilter`, `eth_getFilterChanges`, `eth_getFilterLogs`, `eth_uninstallFilter` | error `-32004` | `backlog` |
| `eth_sendTransaction`, `eth_sign`, `eth_signTransaction`, `eth_coinbase` | error `-32004` | `intentionally-absent` |
| `eth_getStorageAt`, `eth_getStorageValues`, `eth_getProof`, `eth_createAccessList`, `eth_getBlockAccessList`, `eth_simulateV1`, `eth_fillTransaction`, `eth_baseFee`, `eth_blobBaseFee` | error `-32004` | `n/a-by-design` |
| `eth_capabilities`, `eth_config` | error `-32004` | `backlog` |
| Any other name, including unknown `midnight_` names and the `debug_`, `trace_`, `txpool_`, `personal_`, `admin_` and mining namespaces | error `-32601` | — |

### Handled inside the wallet

Never reach this surface. See [handled inside the wallet](ENDPOINTS-DETAILS.md#handled-inside-the-wallet).

| Method | Result | Handled by |
|---|---|---|
| `eth_requestAccounts` | address[] | Wallet |
| `wallet_addEthereumChain`, `wallet_switchEthereumChain` | null | Wallet; stores chain 6201837 with NIGHT at 18 decimals |
| `wallet_watchAsset` | boolean | Wallet; verifies `symbol()` and `decimals()` through `eth_call` |
| `personal_sign`, `eth_signTypedData_v4` | DATA signature | Wallet |

## Design notes

Deviations from Ethereum semantics are deliberate: no historical state for native balances, synthetic pre-London headers, constant fee values, a monotonic transaction count in place of a nonce, and receipts whose logs carry the chain-side transaction hash.
