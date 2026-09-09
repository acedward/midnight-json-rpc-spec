# Endpoints

One row per method. **Result** is the JSON-RPC result type, using the encodings in [README §1](README.md#1-conventions). **Midnight data** names what the value is built from, in the vocabulary of README §1–§3, without the reasoning. Each method links to its section in [ENDPOINTS-DETAILS.md](ENDPOINTS-DETAILS.md).

29 methods are served on both HTTP and WebSocket, plus 2 subscription methods on WebSocket only. 21 further spec-defined methods answer `-32004` ([methods not served](ENDPOINTS-DETAILS.md#methods-not-served)). Any other name answers `-32601`.

## Chain and node identity

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_chainId`](ENDPOINTS-DETAILS.md#eth_chainid) | QUANTITY | Constant `0x5EA1ED` |
| [`net_version`](ENDPOINTS-DETAILS.md#net_version) | string | Constant `"6201837"` |
| [`web3_clientVersion`](ENDPOINTS-DETAILS.md#web3_clientversion) | string | Implementation name and version |
| [`net_listening`](ENDPOINTS-DETAILS.md#net_listening) | boolean | Constant `true` |
| [`eth_syncing`](ENDPOINTS-DETAILS.md#eth_syncing) | `false` | Constant `false` |
| [`eth_accounts`](ENDPOINTS-DETAILS.md#eth_accounts) | address[] | Constant `[]` |

## Fees and gas

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_gasPrice`](ENDPOINTS-DETAILS.md#eth_gasprice) | QUANTITY | Constant `0x3b9aca00` |
| [`eth_estimateGas`](ENDPOINTS-DETAILS.md#eth_estimategas) | QUANTITY | Constant `0x5208` |
| [`eth_feeHistory`](ENDPOINTS-DETAILS.md#eth_feehistory) | `{ oldestBlock, baseFeePerGas[], gasUsedRatio[], reward[][] }` | Zero arrays sized from the requested range; indexer head height when `newestBlock` is a tag |
| [`eth_maxPriorityFeePerGas`](ENDPOINTS-DETAILS.md#eth_maxpriorityfeepergas) | QUANTITY | Constant `0x0` |
| [`web3_sha3`](ENDPOINTS-DETAILS.md#web3_sha3) | DATA, 32 bytes | Keccak-256 of the input |

## Blocks

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_blockNumber`](ENDPOINTS-DETAILS.md#eth_blocknumber) | QUANTITY | Indexer head height |
| [`eth_getBlockByNumber`](ENDPOINTS-DETAILS.md#eth_getblockbynumber) | Block \| null | Indexer block at the tag: height, hash, parent hash, author, timestamp, transaction hashes; fixed values for every other header field |
| [`eth_getBlockByHash`](ENDPOINTS-DETAILS.md#eth_getblockbyhash) | Block \| null | Indexer block by hash, same fields |
| [`eth_getBlockTransactionCountByHash`](ENDPOINTS-DETAILS.md#eth_getblocktransactioncountbyhash) | QUANTITY \| null | Length of the indexer block's transaction list |
| [`eth_getBlockTransactionCountByNumber`](ENDPOINTS-DETAILS.md#eth_getblocktransactioncountbynumber) | QUANTITY \| null | Same, by tag |
| [`eth_getBlockReceipts`](ENDPOINTS-DETAILS.md#eth_getblockreceipts) | Receipt[] \| null | Indexer block's transaction list; per transaction, the data of `eth_getTransactionReceipt` |

## Accounts and tokens

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_getBalance`](ENDPOINTS-DETAILS.md#eth_getbalance) | QUANTITY | Sum of unspent NIGHT UTXOs of the bound identity from the UTXO balance store, × 10^12; for a `contract` kind, the NIGHT entry of `contractAction.unshieldedBalances` |
| [`eth_getTransactionCount`](ENDPOINTS-DETAILS.md#eth_gettransactioncount) | QUANTITY | Count of transaction-index rows sent by the address |
| [`eth_getCode`](ENDPOINTS-DETAILS.md#eth_getcode) | DATA | Registry kind of the address |
| [`eth_call`](ENDPOINTS-DETAILS.md#eth_call) | DATA, ABI-encoded | By registry kind of `to`: `contract` → contract ledger state from `contractAction.state`; `token-unshielded` → UTXO balance store; `token-shielded` → metadata only; `protocol` DUST → dust generation data at call time. Symbol, name and decimals from the token manifest |

## Transactions and receipts

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_getTransactionByHash`](ENDPOINTS-DETAILS.md#eth_gettransactionbyhash) | Transaction \| null | Transaction-index row: sender, recipient, sent-count nonce; indexer block containing the Midnight hash for position |
| [`eth_getTransactionReceipt`](ENDPOINTS-DETAILS.md#eth_gettransactionreceipt) | Receipt \| null | Indexer transaction result, segment results and fee; log-store rows keyed by the Midnight hash |
| [`eth_getTransactionByBlockHashAndIndex`](ENDPOINTS-DETAILS.md#eth_gettransactionbyblockhashandindex) | Transaction \| null | Indexer block transaction list at the index; transaction-index row for sender, recipient and nonce |
| [`eth_getTransactionByBlockNumberAndIndex`](ENDPOINTS-DETAILS.md#eth_gettransactionbyblocknumberandindex) | Transaction \| null | Same, by tag |

## Logs and subscriptions

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_getLogs`](ENDPOINTS-DETAILS.md#eth_getlogs) | Log[] | Log-store rows: paired unshielded Spend/Receive contract events as `Transfer`, unpaired ones as mint or burn, every other event type under its `Midnight<Type>()` topic |
| [`eth_subscribe`](ENDPOINTS-DETAILS.md#eth_subscribe) (WebSocket) | QUANTITY subscription id | `logs`: log-store rows as they commit; `newHeads`: indexer head |
| [`eth_unsubscribe`](ENDPOINTS-DETAILS.md#eth_unsubscribe) (WebSocket) | boolean | Subscription table |

## Write path and discovery

| Endpoint | Result | Midnight data |
|---|---|---|
| [`eth_sendRawTransaction`](ENDPOINTS-DETAILS.md#eth_sendrawtransaction) | DATA, 32 bytes | Relayer's eth-side hash for the forwarded raw transaction |
| [`midnight_getTokenBalances`](ENDPOINTS-DETAILS.md#midnight_gettokenbalances) | `{ address, tokenBalances[], pageKey? }` | UTXO balance store, contract ledger state, dust generation data; metadata from the token manifest |
| [`rpc.discover`](ENDPOINTS-DETAILS.md#rpcdiscover) | OpenRPC document | Configuration |

## Methods not served

Each answers `-32004` with `data: { method, classification, reason, documentation }`. See [methods not served](ENDPOINTS-DETAILS.md#methods-not-served).

| Methods | Result | Classification |
|---|---|---|
| `eth_newFilter`, `eth_newBlockFilter`, `eth_newPendingTransactionFilter`, `eth_getFilterChanges`, `eth_getFilterLogs`, `eth_uninstallFilter` | error `-32004` | `backlog` |
| `eth_sendTransaction`, `eth_sign`, `eth_signTransaction`, `eth_coinbase` | error `-32004` | `intentionally-absent` |
| `eth_getStorageAt`, `eth_getStorageValues`, `eth_getProof`, `eth_createAccessList`, `eth_getBlockAccessList`, `eth_simulateV1`, `eth_fillTransaction`, `eth_baseFee`, `eth_blobBaseFee` | error `-32004` | `n/a-by-design` |
| `eth_capabilities`, `eth_config` | error `-32004` | `backlog` |
| Any other name, including unknown `midnight_` names and the `debug_`, `trace_`, `txpool_`, `personal_`, `admin_` and mining namespaces | error `-32601` | — |

## Handled inside the wallet

Never reach this surface. See [handled inside the wallet](ENDPOINTS-DETAILS.md#handled-inside-the-wallet).

| Method | Result | Handled by |
|---|---|---|
| `eth_requestAccounts` | address[] | Wallet |
| `wallet_addEthereumChain`, `wallet_switchEthereumChain` | null | Wallet; stores chain 6201837 with NIGHT at 18 decimals |
| `wallet_watchAsset` | boolean | Wallet; verifies `symbol()` and `decimals()` through `eth_call` |
| `personal_sign`, `eth_signTypedData_v4` | DATA signature | Wallet |
