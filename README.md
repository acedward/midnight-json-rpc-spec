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

Terms used throughout. A parameter that violates its encoding is answered `-32602`.

| Term | Definition |
|---|---|
| **Chain id** | `6201837`, hex `0x5EA1ED`. `eth_chainId` returns the hex form, `net_version` the decimal string. The id is registered in the ethereum-lists chains registry with native currency NIGHT. |
| **Native currency** | NIGHT, presented with 18 decimals. Midnight amounts are denominated in STAR, 1 NIGHT = 10^6 STAR, and are multiplied by 10^12 at this boundary. The scale is lossless. DUST is denominated in SPECK, 1 DUST = 10^15 SPECK, and is reported in SPECK with 15 decimals. |
| **QUANTITY** | An integer as `0x`-prefixed hex with no leading zeros; zero is `0x0`. Block numbers, balances, gas, nonces, indexes and subscription ids are QUANTITY. Inputs with leading zeros, decimal digits only, or a missing prefix are `-32602`. |
| **DATA** | A byte string as `0x`-prefixed hex, two characters per byte, leading zeros kept, lowercase on output. Length is fixed by meaning: addresses are 20 bytes, hashes and topics 32 bytes. A DATA input of the wrong length is `-32602`. |
| **Block tag** | `latest`, `pending`, `safe` and `finalized` all name the indexer head; Midnight has no fork choice that separates them. `earliest` is height 0. Anything else is a QUANTITY height no greater than 2^31 - 1. |
| **Midnight hex** | Identities, hashes and token types are unprefixed hex on the Midnight side and gain their `0x` prefix only at this boundary. |
| **Surfaces** | HTTP on port 8545 and WebSocket on port 10021. Both serve every method; the WebSocket surface additionally serves `eth_subscribe` and `eth_unsubscribe`. Both apply the same envelope rules ([Transport and envelope](ENDPOINTS-DETAILS.md#transport-and-envelope)). |
| **Error codes** | `-32700` parse · `-32600` invalid request · `-32601` unknown method · `-32602` invalid params · `-32603` internal · `-32004` method known and not served, with `data` · `-32005` limit exceeded. Every error a handler raises deliberately carries one of these codes; anything else is sanitized to `-32603`. |
| **Stores** | The implementation maintains four stores: the **registry** of EVM addresses (§2), the **UTXO balance store** of unspent unshielded value per identity and token type, the **transaction index** of transactions with mapped sender and recipient, and the **log store** of EVM-shaped logs derived from contract events. |
| **Token manifest** | Configuration listing every token the surface serves, with its address kind, symbol, name, decimals and, for contract tokens, the compiled ledger API. It is the single metadata source for `eth_call`, `midnight_getTokenBalances` and the log store. |

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
| **NIGHT** (unshielded, zero color) | Unshielded UTXO ledger. The owner is an unshielded address from the NightExternal key role. | The indexer's `unshieldedTransactions(address)` subscription streams created and spent UTXOs with token type and value; the implementation folds them into the UTXO balance store. A contract's NIGHT is the NIGHT entry of `contractAction(address).unshieldedBalances`. | `eth_getBalance` × 10^12 as the native asset. | A registry binding from the wallet's address to the Midnight identity. |
| **DUST** (tag dust, no color) · needs keys | Dust ledger: outputs with commitments and nullifiers, generated over time by NIGHT UTXOs flagged for dust generation and decaying toward a cap set by the ledger's NIGHT-to-DUST ratio. Denominated in SPECK. | Generation is public per dust address: the `dustGenerations(dustAddress, …)` subscription yields each output's SPECK value at creation, its backing NIGHT UTXO and dtime updates; `dustLedgerEvents` streams initial outputs, spends and parameter changes. Spends are keyed by nullifier, and `dustNullifierTransactions(prefixes)` expects the caller to derive nullifiers from the dust secret key. Balance at time *t* = generated capacity at *t* minus spends. | Virtual ERC-20 at `0x1111…1111`. `balanceOf` is the generated capacity at the time of the call minus the spends the holder has reported; without reported spends it is an upper bound and is labelled as such in the token name. Non-transferable. | A binding to a dust address, and either the dust secret key client-side or client-reported nullifiers for an exact figure. |
| **Native unshielded tokens** (colors, tag unshielded) | The same unshielded UTXO ledger. Color = `rawTokenType(domainSep, contract)`, minted with `mintUnshieldedToken`. | The same `unshieldedTransactions` stream, folded into the UTXO balance store per identity and token type for every color. Contract holdings from `unshieldedBalances`. Activity from `Transaction.unshieldedCreatedOutputs` and `unshieldedSpentOutputs` and from paired Spend/Receive contract events. | One virtual ERC-20 per color at its `token-unshielded` address; `balanceOf` from the store, `totalSupply` the store-wide sum; Transfer logs carry the color address. | The same registry binding as NIGHT. |
| **Native shielded tokens** (colors, tag shielded) · needs keys | Zswap pool: coin commitments in the global Merkle tree, ciphertexts encrypted to the recipient's encryption key, spends by nullifier. The raw color equals the unshielded variant's. | Both indexer paths need keys. `zswapLedgerEvents(id)` is a firehose replayed client-side with the Zswap secret keys. `connect(viewingKey)` followed by `shieldedTransactions(sessionId)` yields received outputs server-side, but nullifiers need the coin secret key, so spends stay invisible to a viewing-key holder. | The shared surface answers metadata only. Balances are served either client-side, by a wallet extension holding the Zswap keys, or by a per-user session surface bound to a viewing key and fed client-reported nullifiers, at the color's `token-shielded` address. | Zswap secret keys client-side, or a viewing key plus reported nullifiers on a per-user surface. |
| **Ledger unshielded** (contract account model) | The contract's public ledger fields, e.g. a map of holder to `Uint<128>`. Ethereum-native holders appear as the 20-byte address zero-padded. | `contract(address).state` or `contractAction(address, offset).state` returns the serialized ledger state. It is read either by executing the contract's read circuit locally against that state with the compiled contract module, or by decoding the ledger field with the module's `ledger(state)` accessor. Transfer logs from `contractEvents` give the same balances by fold and serve activity. | `eth_call` ERC-20 views at the contract's address; Transfer logs through `eth_getLogs`. | The contract's compiled module in the token manifest. |
| **Ledger shielded** (contract account model, private) · needs keys | Two cases. (a) Shielded coins the contract itself holds, recorded in its public ledger as `QualifiedShieldedCoinInfo` and in `contractAction.zswapState`. (b) Holder balances the contract keeps private: the ledger stores commitments; values live in the holder's private state in the dapp. | (a) The same state read; values are visible. (b) Only the holder can compute it, through the dapp's private-state provider or a wallet extension; a contract may disclose through its own events. | (a) `balanceOf(contractAddress)` on the color's shielded ERC-20 view. (b) Client-side only; the shared surface answers `0x`. | (a) Nothing beyond the compiled module. (b) The holder's private state. |

## 4. Endpoints

- [ENDPOINTS.md](ENDPOINTS.md) lists every method with its result type and the Midnight data behind it.
- [ENDPOINTS-DETAILS.md](ENDPOINTS-DETAILS.md) defines parameters, behaviour and errors per method, the [methods not served](ENDPOINTS-DETAILS.md#methods-not-served), the [transport and envelope rules](ENDPOINTS-DETAILS.md#transport-and-envelope), the [wallet-side methods](ENDPOINTS-DETAILS.md#handled-inside-the-wallet) that shape this surface, and the [conformance checks](ENDPOINTS-DETAILS.md#conformance-checks) an implementation must pass.

## Design notes

Deviations from Ethereum semantics are deliberate consequences of Midnight not being an EVM chain: no historical state for native balances, synthetic pre-London headers, constant fee values, a monotonic transaction count in place of a nonce, and receipts whose logs carry the chain-side transaction hash. A pre-London header is the block header shape from before Ethereum's London fork: it carries no `baseFeePerGas`, which is what makes wallets build legacy type-0 transactions.
