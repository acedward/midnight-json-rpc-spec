# Midnight EVM JSON-RPC Specification

**Status:** Draft · 2026-09-08

A JSON-RPC 2.0 surface that lets Ethereum wallets and tooling read Midnight and submit signed transactions to it. This repository is a specification, not an implementation. Any system that serves the methods listed in [ENDPOINTS.md](ENDPOINTS.md) with the behaviour defined in [ENDPOINTS-DETAILS.md](ENDPOINTS-DETAILS.md) conforms.

The surface is not an Ethereum node. There is no EVM execution, no storage trie and no mempool. Every result is derived from the Midnight indexer, from stores the implementation maintains, from configuration, or from a relayer. Behaviour not described here is out of scope.

## Reading order

| File | Contents |
|---|---|
| [README.md](README.md) | Key concepts: conventions, addresses, balance kinds |
| [ENDPOINTS.md](ENDPOINTS.md) | One row per endpoint: result type and the Midnight data behind it |
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
| **Stores** | The implementation maintains four stores: the **registry** of EVM addresses (§2), the **UTXO balance store** of unspent unshielded value per identity and token type, the **transaction index** of transactions with mapped sender and recipient, and the **log store** of EVM-shaped logs derived from contract events. |
| **Token manifest** | Configuration listing every token the surface serves: address kind, symbol, name, decimals and, for contract tokens, the compiled contract module. It is the single metadata source for `eth_call`, `midnight_getTokenBalances` and the log store. |

## 2. Addresses

Midnight identities are 32 bytes; EVM addresses are 20. Every EVM address the surface answers for is derived by one of five rules and recorded in the registry with its kind. The registry is the discriminator: every read dispatches on the kind of the address it was asked about, never on the shape of the address.

| Kind | Derivation | Registry kind | Rules |
|---|---|---|---|
| **Midnight user identity** | `keccak256(identity)[12:32]` of the 32-byte identity | `midnight` | Registered at first sighting. `eth_getCode` answers `0x`. |
| **Ethereum-native identity** | The 20 bytes verbatim. On the Midnight side such an identity travels as the address zero-left-padded to 32 bytes; a 32-byte identity whose first 12 bytes are zero is read as this kind | `ethereum` | The embedding is lossless. Midnight-side balances are attributed to an Ethereum-native account only through a registry binding to a Midnight identity; how bindings are established is outside this specification. |
| **Compact contract** | `keccak256(contractAddress)[12:32]` of the 32-byte contract address | `contract` | The contract's EVM address is the ERC-20 address for balances the contract manages in its own ledger state. It is also the holder address for value the contract owns. |
| **Token color** | `keccak256("midnight-evm:token:" ‖ tag ‖ raw)[12:32]`, where `raw` is the 32-byte token type and `tag` is the ASCII pool tag, `unshielded` or `shielded` | `token-unshielded`, `token-shielded` | A raw color is the same value in both pools; the tag distinguishes them, so it is part of the derivation. A color minted shielded and unshielded is two ERC-20s. A contract minting two colors is two ERC-20s, neither of which is the contract's own address. |
| **Protocol asset** | A fixed constant. **DUST** is `0x1111111111111111111111111111111111111111` | `protocol` | DUST has no raw color, so a constant is the only possible address. Further protocol assets take further constants of the same form. |

**Registry rules.** The registry is unique on EVM address. Two identities deriving the same address is a hard error that aborts the write; it is never a merge. Every kind other than `midnight` and `ethereum` answers the code marker in `eth_getCode`. Token metadata comes from the token manifest.

**Argument translation.** An ERC-20 argument arrives as a 20-byte address and is translated to the Compact type through the registry: an `ethereum` kind becomes the address zero-left-padded in the user branch, a `midnight` kind becomes its stored 32-byte identity, a `contract` kind goes into the contract branch. Calldata is never forwarded into a circuit unchanged.

## 3. Balance kinds

Six kinds of value exist on Midnight and they are read from different places. The wallet-facing method is `eth_getBalance` for NIGHT and `eth_call` for everything else. Where a row says **needs keys**, the number cannot be computed without the holder's secret material and the shared surface does not compute it.

| Balance | Where it lives | Source | Exposure | Requires |
|---|---|---|---|---|
| **NIGHT** (unshielded, zero color) | Unshielded UTXO ledger; owner is an unshielded address. | `unshieldedTransactions(address)`, folded into the UTXO balance store. A contract's NIGHT: the NIGHT entry of `contractAction(address).unshieldedBalances`. | `eth_getBalance`, × 10^12. | A registry binding from the wallet's address to the Midnight identity. |
| **DUST** (tag dust, no color) · needs keys | Dust ledger; generated by NIGHT UTXOs registered for dust generation, decaying toward a cap. SPECK, 15 decimals. | Generation per dust address from the `dustGenerations(dustAddress, …)` subscription and `dustLedgerEvents`. Spends are keyed by nullifier via `dustNullifierTransactions(prefixes)`, which the holder derives from the dust secret key. Balance at *t* = generated capacity at *t* minus spends. | Virtual ERC-20 at `0x1111…1111`. `balanceOf` is generated capacity at call time minus reported spends; without reported spends it is an upper bound and the token name says so. Non-transferable. | A binding to a dust address; the dust secret key client-side or client-reported nullifiers for an exact figure. |
| **Native unshielded tokens** (colors, tag unshielded) | Unshielded UTXO ledger; color = `rawTokenType(domainSep, contract)`. | The same `unshieldedTransactions` stream, folded per identity and token type. Contract holdings from `unshieldedBalances`. Activity from `Transaction.unshieldedCreatedOutputs` / `unshieldedSpentOutputs` and paired Spend/Receive contract events. | One virtual ERC-20 per color at its `token-unshielded` address; `balanceOf` from the store, `totalSupply` the store-wide sum; Transfer logs carry the color address. | The same registry binding as NIGHT. |
| **Native shielded tokens** (colors, tag shielded) · needs keys | Zswap pool; same raw color as the unshielded variant; spends by nullifier. | `zswapLedgerEvents(id)` replayed client-side with the Zswap secret keys, or `connect(viewingKey)` + `shieldedTransactions(sessionId)` for received outputs only; spends need the coin secret key. | Shared surface answers metadata only. Balances are served client-side by a wallet extension holding the keys, or by a per-user session surface bound to a viewing key and fed client-reported nullifiers, at the `token-shielded` address. | Zswap secret keys client-side, or a viewing key plus reported nullifiers on a per-user surface. |
| **Ledger unshielded** (contract account model) | The contract's public ledger fields, e.g. `Map<holder, Uint<128>>`; Ethereum-native holders as the 20-byte address zero-padded. | `contract(address).state` or `contractAction(address, offset).state`, read by executing the contract's read circuit locally against that state or by decoding the ledger field with the compiled module's `ledger(state)` accessor. Transfer logs from `contractEvents` give the same balances by fold and serve activity. | `eth_call` ERC-20 views at the contract's address; Transfer logs through `eth_getLogs`. | The contract's compiled module in the token manifest. |
| **Ledger shielded** (contract account model, private) · needs keys | (a) Shielded coins the contract holds, in its ledger as `QualifiedShieldedCoinInfo` and in `contractAction.zswapState`. (b) Holder balances kept as commitments; values live in the holder's private state. | (a) The same state read. (b) Only the holder, through the dapp's private state or a wallet extension; a contract may disclose through its own events. | (a) `balanceOf(contractAddress)` on the color's shielded ERC-20 view. (b) Client-side only; the shared surface answers `0x`. | (a) The compiled module. (b) The holder's private state. |

## 4. Endpoints

- [ENDPOINTS.md](ENDPOINTS.md) lists every method with its result type and the Midnight data behind it.
- [ENDPOINTS-DETAILS.md](ENDPOINTS-DETAILS.md) defines parameters, behaviour and errors per method, the [methods not served](ENDPOINTS-DETAILS.md#methods-not-served), the [transport and envelope rules](ENDPOINTS-DETAILS.md#transport-and-envelope), the [wallet-side methods](ENDPOINTS-DETAILS.md#handled-inside-the-wallet) that shape this surface, and the [conformance checks](ENDPOINTS-DETAILS.md#conformance-checks) an implementation must pass.

## Design notes

Deviations from Ethereum semantics are deliberate: no historical state for native balances, synthetic pre-London headers, constant fee values, a monotonic transaction count in place of a nonce, and receipts whose logs carry the chain-side transaction hash.
