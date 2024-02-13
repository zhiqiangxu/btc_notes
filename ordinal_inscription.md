<!-- omit in toc -->
# Ordinal Inscription

**Table of Contents**

- [Introduction](#introduction)
- [Implementation](#implementation)
  - [Ordinal Theory](#ordinal-theory)
  - [Inscription protocol](#inscription-protocol)
    - [Inscribe an inscription](#inscribe-an-inscription)
    - [Transfer an inscription](#transfer-an-inscription)


# Introduction

"Ordinal Inscription" is essentially a 2-step protocol:
1. It defines a "Ordinal Theory" which defines an order for each satosh of utxo.
   1. Satoshis are numbered in the order in which they're mined, and transferred from transaction inputs to transaction outputs first-in-first-out.
2. It defines an "inscription protocol" which allows inscribe a utxo satosh with arbitrary content, and the inscription can be transfered by transfering the corresponding satosh.
   1. The inscription content is stored in taproot[^1] script-path spend scripts, thus inscriptions are made using a two-phase commit/reveal procedure.


# Implementation

Now let's go through the details of the above mentioned "Ordinal Theory" and "inscription protocol".

## Ordinal Theory

It's implemented at both tx level and block level.

1. At tx level, the main logic is in [`index_transaction_sats`](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L643).
     1. Each tx input contains precomputed sat ranges, the [`input_sat_ranges`](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L648) are [assigned](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L684) to outputs first-in-first-out, the sat ranges assigned to each output are [stored](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L729) to serve as precomputed sat ranges when spent as tx input.
2. At block level: tx fees are [appended](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L507) to the tail of [coinbase input sat ranges](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L460), burned sats are [assigned](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L546) to [`null` output](https://github.com/rust-bitcoin/rust-bitcoin/blob/8f7cc4d6b358bc28fcc01098df7270247d331f2a/bitcoin/src/blockdata/transaction.rs#L84-L88), thus `index_transaction_sats` is first [applied](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L496) on normal transactions, then [applied](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L511) on coinbase tx.

## Inscription protocol

The overall process is essentially very similar to [`index_transaction_sats`](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L643), the main difference is:

1. At tx level, the role of [`input_sat_ranges`](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L648) for [`index_transaction_sats`](https://github.com/ordinals/ord/blob/69925f12710d3ed92145c97997039ea280594fa5/src/index/updater.rs#L643) is taken by [`floating_inscriptions`](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/index/updater/inscription_updater.rs#L77) instead, which contains both [previously inscribed](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/index/updater/inscription_updater.rs#L95) inscriptions and inscriptions [being inscribed](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/index/updater/inscription_updater.rs#L136-L222). 
2. At block level, inscriptions on tx fee sats are saved in the [`flotsam`](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/index/updater/inscription_updater.rs#L44) field, and later [appended](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/index/updater/inscription_updater.rs#L272) to the tail of `floating_inscriptions` for coinbase tx, inscriptions on burned sats are [assigned](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/index/updater/inscription_updater.rs#L339-L345) to the corresponding sat point of the null output.

In order to inscribe an inscription on a specific tx input, just ensure the serialized inscription content is contained within the witness [tapscript](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/inscriptions/envelope.rs#L105).
By default the inscription is inscribed on the first sat of its input, unless it has a [pointer tag](https://docs.ordinals.com/inscriptions/pointer.html) specified to explicitly change the position.

Inscription content is serialized using data pushes within unexecuted conditionals of tapscript, called "envelopes". Envelopes consist of an `OP_FALSE` `OP_IF` … `OP_ENDIF` wrapping any number of data pushes.

A text inscription containing the string "Hello, world!" is serialized as follows:

```
OP_FALSE
OP_IF
  OP_PUSH "ord"
  OP_PUSH 1
  OP_PUSH "text/plain;charset=utf-8"
  OP_PUSH 0
  OP_PUSH "Hello, world!"
OP_ENDIF
```

First the string `ord` is pushed, to disambiguate inscriptions from other uses of envelopes, checked [here](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/inscriptions/envelope.rs#L156).

`OP_PUSH 1` encodes the content type tag, which indicates that the next push contains the content type, and `OP_PUSH 0` indicates that subsequent data pushes contain the content itself. 

The general process for serializing inscription content is: [first](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/inscriptions/envelope.rs#L44) the tag part is serialized as tag_id/value pairs, [then](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/inscriptions/envelope.rs#L32-L36) the body part is serialized as `OP_PUSH 0` followed by data pushes. A complete list of valid tags can be checked [here](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/inscriptions/tag.rs#L24-L37), with explanations [here](https://docs.ordinals.com/inscriptions.html#fields).



### Inscribe an inscription

Follow the `ord wallet inscribe` command [here](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/subcommand/wallet/inscribe.rs#L74-L170) to inscribe an inscription to some destination address. 

Basically users only need to specify the fee rate, the inscription content, the destination address and the optional sat point, then [`create_batch_inscription_transactions`](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/wallet/inscribe/batch.rs#L176) will finally be called to generate proper commit/reveal transactions, tweaked BIP-340 recovery key for tapscript.

### Transfer an inscription

Follow the `ord wallet send` command [here](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/subcommand/wallet/send.rs#L32-L107) to send inscription to some destination address.

Basically users only need to specify the fee rate, the inscription id, the destination address and the postage to include with the sent inscription, then [`create_unsigned_send_satpoint_transaction`](https://github.com/ordinals/ord/blob/989b763afa4d61c4b03e224e175446d055ca4741/src/subcommand/wallet/send.rs#L167) will be called to generate a single tx that transfers the target sat point to the destination address.



[^1]: This post assumes knowledge of the Bitcoin taproot.