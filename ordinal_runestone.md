<!-- omit in toc -->
# Ordinal Runestone

**Table of Contents**
- [Introduction](#introduction)
- [Implementation](#implementation)
  - [Issue tokens](#issue-tokens)
  - [Mint tokens](#mint-tokens)
  - [Transfer tokens](#transfer-tokens)
  - [Burn tokens](#burn-tokens)
- [Difference with Inscriptions](#difference-with-inscriptions)


# Introduction

[Runestone](https://github.com/casey/runestone) is a fungible token protocol for Bitcoin.

It allows to issue/mint/transfer/burn tokens in a utxo[^1] way by client side validation, which means the Bitcoin itself is not aware of these tokens.

The token owner must be some non-[`OP_RETURN`](https://github.com/rust-bitcoin/rust-bitcoin/blob/a3c4194c3fc76778bc4af49f38b3a64d00b5fc49/bitcoin/src/blockdata/script/borrowed.rs#L349) tx output, tokens allocated to `OP_RETURN` output are considered burned.


The overall process is: 

In order to spend(transfer/burn) tokens, the user just spends the corresponding tx output, but the spending details are encoded into the [`Runestone`](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/runes/runestone.rs#L6-L11) structure in a `OP_RETURN` output. 

In order to issue/mint tokens, the user also has to encode the intent into the `Runestone` structure in a `OP_RETURN` output, but this time no need to spend any tx output.

A single tx can only host a single `RuneStone` object, only the [first](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/runes/runestone.rs#L209-L229) matching tx output will be effective.


# Implementation

Here's the above mentioned [`Runestone`](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/runes/runestone.rs#L6) structure:

```rust
pub struct Runestone {
  pub edicts: Vec<Edict>,
  pub etching: Option<Etching>,
  pub default_output: Option<u32>,
  pub burn: bool,
}
```

1. The `etching` field is used for issuing tokens.
2. The `edicts` field is used for minting/transfering tokens.
3. The `default_output` field is used for specifying a default output for "`unallocated`" tokens.
4. The `burn` field is used for burning tokens.

"`unallocated`" deserves some more explanations here. 

The tx input specifies the tokens that the user wants to spend, these tokens are [initially](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/index/updater/rune_updater.rs#L47-L65) considered `unallocated` unless allocated by [`RuneStone.edicts`](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/index/updater/rune_updater.rs#L205-L209).

Let's go through the implementation details of each operation, more specically: issue/mint/transfer/burn tokens.

## Issue tokens

We'll need to make use of the `Runestone.etching` field above, let's take a closer look at the `Etching` structure:

```rust
pub struct Etching {
  pub divisibility: u8,
  pub mint: Option<Mint>,
  pub rune: Option<Rune>,
  pub spacers: u32,
  pub symbol: Option<char>,
}

pub struct Mint {
  pub deadline: Option<u32>,
  pub limit: Option<u128>,
  pub term: Option<u32>,
}

pub struct Rune(pub u128);
```

1. `divisibility` represents the precision of amount, corresponding to `decimals` in ERC20.
2. `mint` contains 3 optional parameters for minting:
    1. `deadline` configures the deadline timestamp if specified.
    2. `limit` configures the max amount of token each single tx can mint if specified.
    3. `term` configures the termination block if specified.
3. `rune` is an optional id for the token, an internal id will be [automatically generated](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/index/updater/rune_updater.rs#L100) if it's `None`.
    1. The generated id won't conflict with user provided id because of [this check](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/index/updater/rune_updater.rs#L100).
4. `spacers` is the bit positions of "•" characters, follow [here](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/runes/spaced_rune.rs#L128-L129) for an example.
5. `symbol` is an optional single character symbol for this token.


Here's an example to issue an token:

```rust
let runestone = Runestone {
    etching: Some(Etching {
    divisibility: 3u8,
    mint: None,
    rune: None,
    spacers:0,
    symbol: Some('$'),
    }),
    ..Default::default()
};
let script_pubkey = runestone.encipher();

ensure!(
    script_pubkey.len() <= 82,
    "runestone greater than maximum OP_RETURN size: {} > 82",
    script_pubkey.len()
);

let unfunded_transaction = Transaction {
    version: 2,
    lock_time: LockTime::ZERO,
    input: Vec::new(),
    output: vec![
        TxOut {
            script_pubkey,
            value: 0,
        },
        TxOut {
            script_pubkey: destination.script_pubkey(),
            value: TARGET_POSTAGE.to_sat(),
        },
    ],
};

let inscriptions = wallet
    .get_inscriptions()?
    .keys()
    .map(|satpoint| satpoint.outpoint)
    .collect::<Vec<OutPoint>>();

if !bitcoin_client.lock_unspent(&inscriptions)? {
    bail!("failed to lock UTXOs");
}

let unsigned_transaction =
    fund_raw_transaction(&bitcoin_client, self.fee_rate, &unfunded_transaction)?;

let signed_transaction = bitcoin_client
    .sign_raw_transaction_with_wallet(&unsigned_transaction, None, None)?
    .hex;

let transaction = bitcoin_client.send_raw_transaction(&signed_transaction)?;
```
Basically one only needs to tweak the runestone object, all other codes are template codes copied from [here](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/subcommand/wallet/etch.rs#L60-L118).

The canonical `id` of the issued token is computed [here](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/index/updater/rune_updater.rs#L130), which uniquely locates the etching tx with block height and tx index within the block.



## Mint tokens

We'll need to make use of the `Runestone.edicts` field above, let's take a closer look at the `Edict` structure:

```rust
pub struct Edict {
  pub id: u128,
  pub amount: u128,
  pub output: u128,
}
```

Basically this structure represents the intention to mint some `amount` of the `id` token to some tx `output` of the minting tx.

In order to mint an existing token, one just needs to set `Edict.id` to "`u128::from(id) | CLAIM_BIT`", where `id` is the canonical id of the issued token, refer [here](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/index/updater/rune_updater.rs#L199-L204) for details.

In order to both issue and mint a token at the same time, one needs to make sure to specify a valid `Runestone.etching` and then set `Edict.id` to `0`, refer [here](https://github.com/ordinals/ord/blob/416b286759e2d8b985cefed4d0bd21b91942e266/src/index/updater/rune_updater.rs#L190-L199) for details.

Here's an example to mint an existing token:

```rust
let runestone = Runestone {
    edicts: vec![Edict {
        amount: 10u128,
        id: issued_id  | CLAIM_BIT,
        // mint it to tx output 1
        output: 1,
    }],
    ..Default::default()
};
let script_pubkey = runestone.encipher();
// The remaining logic is the same as before
```

Here's an example to both issue and mint at the same time:

```rust
let runestone = Runestone {
    etching: Some(Etching {
        divisibility: 3u8,
        mint: None,
        rune: None,
        spacers:0,
        symbol: Some('$'),
    }),
    edicts: vec![Edict {
        amount: 10u128,
        // mint the issuing token
        id: 0,
        // mint it to tx output 1
        output: 1,
    }],
    ..Default::default()
};
let script_pubkey = runestone.encipher();
// The remaining logic is the same as before
```

## Transfer tokens

The transfer function is coerced into `Runestone.edicts` field.

In order to transfer some token, the user has to spend the corresponding tx output and allocate the tokens with `Runestone.edicts`.

Unallocated tokens are either allocated to the tx output specified by `Runestone.default_output` if it's a non `OP_RETURN` output, or the first non `OP_RETURN` output if there is no default, or burned otherwise.

Here's an example to transfer tokens from some tx output:

```rust
let runestone = Runestone {
    edicts: vec![Edict {
        amount: 10u128,
        id: id_to_transfer,
        // mint it to tx output 1
        output: 1,
    }],
    ..Default::default()
};
let script_pubkey = runestone.encipher();
ensure!(
    script_pubkey.len() <= 82,
    "runestone greater than maximum OP_RETURN size: {} > 82",
    script_pubkey.len()
);

let unfunded_transaction = Transaction {
    version: 2,
    lock_time: LockTime::ZERO,
    input: vec![
        tx_input_to_spend,
    ],
    output: vec![
        TxOut {
            script_pubkey,
            value: 0,
        },
        TxOut {
            script_pubkey: destination.script_pubkey(),
            value: TARGET_POSTAGE.to_sat(),
        },
    ],
};

// The remaining logic is the same as before
```
Basically it's very similar to mint an existing token, except that one has to spend the tx output by specifying corresponding `tx_input_to_spend` as tx input.

## Burn tokens

In order to burn tokens, one just needs to set `Runestone.burn` to true and spend the corresponding tx output.

Here's an example to burn tokens from some tx output:

```rust
let runestone = Runestone {
    burn: true,
    ..Default::default()
};
let script_pubkey = runestone.encipher();
ensure!(
    script_pubkey.len() <= 82,
    "runestone greater than maximum OP_RETURN size: {} > 82",
    script_pubkey.len()
);

let unfunded_transaction = Transaction {
    version: 2,
    lock_time: LockTime::ZERO,
    input: vec![
        tx_input_to_spend,
    ],
    output: vec![
        TxOut {
            script_pubkey,
            value: 0,
        },
    ],
};

// The remaining logic is the same as before
```

 Basically it's very similar to transfer tokens in that one needs to spend the corresponding tx output, but no need for `Runestone.edicts`.

# Difference with Inscriptions

1. No dependency for taproot.
2. Each operation only needs `1` tx on Bitcoin.


[^1]: This post assumes knowledge of the Bitcoin utxo.