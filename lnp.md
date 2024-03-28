

# HTLC script

There're two flavors of HTLC script: segwit and taproot. 

For each flavor, there're two versions: receiver's and sender's version.
(This is in order to ascribe blame for commitment transactions.)

For each version, there're two cases depending on whether the local party is a receiver or a sender.

In general, each script has to deal with 3 cases:

1. The success case: the receiver claims the funds.
2. The timeout case: the sender claims back the funds after timeout.
3. The revocation case: the counter party sweeps all the funds if a breached commitment was broadcast.

For taproot flavor, only the first 2 cases are handled by script, the revocation case is handled by using the revocation key as internal key of taproot output, [source](https://github.com/lightningnetwork/lnd/blob/15c7686830934dc6deb063e9a4f014a76ed7649d/input/script_utils.go#L745).

(To spend a taproot output, either provide one of the scripts or generate a signature corresponding to the `taprootKey` computed [here](https://github.com/btcsuite/btcd/blob/8b2f43e9cee2b90e5456c2f928386832a4ca51f2/txscript/taproot.go#L244).)


## Script construction:

1. [genSegwitV0HtlcScript](https://github.com/lightningnetwork/lnd/blob/b575460bca291681364e55ffb8e553ef73db920b/lnwallet/commitment.go#L1004)
   1. [ReceiverHTLCScript](https://github.com/lightningnetwork/lnd/blob/b575460bca291681364e55ffb8e553ef73db920b/input/script_utils.go#L961)

    ```golang
    // ReceiverHTLCScript constructs the public key script for an incoming HTLC
    // output payment for the receiver's version of the commitment transaction. The
    // possible execution paths from this script include:
    //   - The receiver of the HTLC uses its second level HTLC transaction to
    //     advance the state of the HTLC into the delay+claim state.
    //   - The sender of the HTLC sweeps all the funds of the HTLC as a breached
    //     commitment was broadcast.
    //   - The sender of the HTLC sweeps the HTLC on-chain after the timeout period
    //     of the HTLC has passed.
    //
    // If confirmedSpend=true, a 1 OP_CSV check will be added to the non-revocation
    // cases, to allow sweeping only after confirmation.
    //    
    // Possible Input Scripts:
    //
    //	RECVR: <0> <sender sig> <recvr sig> <preimage> (spend using HTLC success transaction)
    //	REVOK: <sig> <key>
    //	SENDR: <sig> 0
    //
    // Received HTLC Output Script:
    //
    //	 OP_DUP OP_HASH160 <revocation key hash160> OP_EQUAL
    //	 OP_IF
    //	 	OP_CHECKSIG
    //	 OP_ELSE
    //		<sendr htlc key>
    //		OP_SWAP OP_SIZE 32 OP_EQUAL
    //		OP_IF
    //		    OP_HASH160 <ripemd160(payment hash)> OP_EQUALVERIFY
    //		    2 OP_SWAP <recvr htlc key> 2 OP_CHECKMULTISIG
    //		OP_ELSE
    //		    OP_DROP <cltv expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
    //		    OP_CHECKSIG
    //		OP_ENDIF
    //		[1 OP_CHECKSEQUENCEVERIFY OP_DROP] <- if allowing confirmed
    //		spend only.
    //	 OP_ENDIF
    ```

   2. [SenderHTLCScript](https://github.com/lightningnetwork/lnd/blob/b575460bca291681364e55ffb8e553ef73db920b/input/script_utils.go#L330)

    ```golang
    // SenderHTLCScript constructs the public key script for an outgoing HTLC
    // output payment for the sender's version of the commitment transaction. The
    // possible script paths from this output include:
    //
    //   - The sender timing out the HTLC using the second level HTLC timeout
    //     transaction.
    //   - The receiver of the HTLC claiming the output on-chain with the payment
    //     preimage.
    //   - The receiver of the HTLC sweeping all the funds in the case that a
    //     revoked commitment transaction bearing this HTLC was broadcast.
    //
    // If confirmedSpend=true, a 1 OP_CSV check will be added to the non-revocation
    // cases, to allow sweeping only after confirmation.
    //    
    // Possible Input Scripts:
    //
    //	SENDR: <0> <sendr sig>  <recvr sig> <0> (spend using HTLC timeout transaction)
    //	RECVR: <recvr sig>  <preimage>
    //	REVOK: <revoke sig> <revoke key>
    //	 * receiver revoke
    //
    // Offered HTLC Output Script:
    //
    //	 OP_DUP OP_HASH160 <revocation key hash160> OP_EQUAL
    //	 OP_IF
    //		OP_CHECKSIG
    //	 OP_ELSE
    //		<recv htlc key>
    //		OP_SWAP OP_SIZE 32 OP_EQUAL
    //		OP_NOTIF
    //		    OP_DROP 2 OP_SWAP <sender htlc key> 2 OP_CHECKMULTISIG
    //		OP_ELSE
    //		    OP_HASH160 <ripemd160(payment hash)> OP_EQUALVERIFY
    //		    OP_CHECKSIG
    //		OP_ENDIF
    //		[1 OP_CHECKSEQUENCEVERIFY OP_DROP] <- if allowing confirmed
    //		spend only.
    //	 OP_ENDIF
    ```

5. [genTaprootHtlcScript](https://github.com/lightningnetwork/lnd/blob/b575460bca291681364e55ffb8e553ef73db920b/lnwallet/commitment.go#L1078)
   1. [ReceiverHTLCScriptTaproot](https://github.com/lightningnetwork/lnd/blob/15c7686830934dc6deb063e9a4f014a76ed7649d/input/script_utils.go#L1348)

    ```golang
    // ReceiverHTLCScriptTaproot constructs the taproot witness program (schnor
    // key) for an incoming HTLC on the receiver's version of the commitment
    // transaction. This method returns the top level tweaked public key that
    // commits to both the script paths. From the PoV of the receiver, this is an
    // accepted HTLC.
    //    
    // The returned key commits to a tapscript tree with two possible paths:
    //
    //   - The timeout path:
    //     <remote_htlcpubkey> OP_CHECKSIG
    //     1 OP_CHECKSEQUENCEVERIFY OP_DROP
    //     <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
    //
    //   - Success path:
    //     OP_SIZE 32 OP_EQUALVERIFY
    //     OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
    //     <local_htlcpubkey> OP_CHECKSIGVERIFY
    //     <remote_htlcpubkey> OP_CHECKSIG
    //
    // The timeout path can be spent with a witness of:
    //   - <sender sig> <timeout_script> <control_block>
    //
    // The success path can be spent with a witness of:
    //   - <sender sig> <receiver sig> <preimage> <success_script> <control_block>
    //
    // The top level keyspend key is the revocation key, which allows a defender to
    // unilaterally spend the created output. Both the final output key as well as
    // the tap leaf are returned.
    ```

    2. [SenderHTLCScriptTaproot](https://github.com/lightningnetwork/lnd/blob/15c7686830934dc6deb063e9a4f014a76ed7649d/input/script_utils.go#L789)

    ```golang
    // SenderHTLCScriptTaproot constructs the taproot witness program (schnorr key)
    // for an outgoing HTLC on the sender's version of the commitment transaction.
    // This method returns the top level tweaked public key that commits to both
    // the script paths. This is also known as an offered HTLC.
    //    
    // The returned key commits to a tapscript tree with two possible paths:
    //
    //   - Timeout path:
    //     <local_key> OP_CHECKSIGVERIFY
    //     <remote_key> OP_CHECKSIG
    //
    //   - Success path:
    //     OP_SIZE 32 OP_EQUALVERIFY
    //     OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
    //     <remote_htlcpubkey> OP_CHECKSIG
    //     1 OP_CHECKSEQUENCEVERIFY OP_DROP
    //
    // The timeout path can be spent with a witness of (sender timeout):
    //
    //	<receiver sig> <local sig> <timeout_script> <control_block>
    //
    // The success path can be spent with a valid control block, and a witness of
    // (receiver redeem):
    //
    //	<receiver sig> <preimage> <success_script> <control_block>
    //
    // The top level keyspend key is the revocation key, which allows a defender to
    // unilaterally spend the created output.
    ```

Taproot

1. [AssembleTaprootScriptTree](https://github.com/btcsuite/btcd/blob/8b2f43e9cee2b90e5456c2f928386832a4ca51f2/txscript/taproot.go#L623)
    1. This function assembles each taproot script as a merkle leaf and generates a merkle proof for each leaf.
2. [ComputeTaprootOutputKey](https://github.com/btcsuite/btcd/blob/8b2f43e9cee2b90e5456c2f928386832a4ca51f2/txscript/taproot.go#L244)
    1. This functions calculates a top-level taproot output key given an internal key, and tapscript merkle root. The final key is derived as:

    ```golang
    taprootKey = internalKey + (h_tapTweak(internalKey || merkleRoot)*G)
    ```

3. [PayToTaprootScript](https://github.com/lightningnetwork/lnd/blob/15c7686830934dc6deb063e9a4f014a76ed7649d/input/taproot.go#L143)
   1. This function creates a new script to pay to a version 1 (taproot) witness program. The passed public key will be serialized as an x-only key to create the witness program.


# References
1. https://lightning.network/lightning-network-paper.pdf
2. https://www.derpturkey.com/revocable-transactions-with-ln-penalty/
3. https://learnblockchain.cn/article/5818
4. https://github.com/BlockchainCommons/Learning-Bitcoin-from-the-Command-Line/blob/9b97c50471226fc8fdb6063508c75db156bf5122/11_3_Using_CSV_in_Scripts.md
5. https://github.com/BlockchainCommons/Learning-Bitcoin-from-the-Command-Line/blob/9b97c50471226fc8fdb6063508c75db156bf5122/11_2_Using_CLTV_in_Scripts.md
6. https://medium.com/softblocks/lightning-network-in-depth-part-2-htlc-and-payment-routing-db46aea445a8
7. https://ellemouton.com/posts/htlc-deep-dive/