#################
Cross-Chain Swaps
#################

A cross-chain swap enables **trading tokens** across **different blockchains**, without using an intermediary party (e.g. an exchange service) in the process.

.. figure:: ../resources/images/examples/cross-chain-swap.png
    :align: center
    :width: 400px

    Atomic cross-chain swap between public and private network

In order to create a trustless environment for an exchange, a specific transaction type is required that is commonly referred to as **Hashed TimeLock Contract** (HTLC). Two additional components characterize this transaction type: *hashlocks* and *timelocks*. A thorough explanation can be found on the `Bitcoin Wiki <https://en.bitcoin.it/wiki/Hashed_Timelock_Contracts>`_.

In other words, to reduce counterparty risk, the receiver of a payment needs to present a proof for the transaction to execute. Failing to do so, the locked funds are released after the deadline is reached, even if just one actor does not agree.

********
Protocol
********

Alice and Bob want to exchange **10 alice tokens for 10 bob tokens**. The problem is that they are not in the same blockchain: alice token is defined in NEM public chain, whereas bob token is only present in a private chain using Catapult technology.

.. note:: NEM's private and future public chain share the SDK. You could implement atomic cross-chain swap between blockchains that use different technologies if they permit the :ref:`secret lock/proof mechanism <lock-hash-algorithm>`.

.. mermaid:: ../resources/diagrams/cross-chain-swap.mmd
    :caption: Atomic cross-chain swap sequence diagram
    :align: center

1. Alice generates a random set of bytes called ``proof``. The proof should have a size between ``10`` and ``1000`` bytes.

2. Alice hashes the obtained proof with one of the :ref:`available algorithms <lock-hash-algorithm>` to generate the ``secret``.

3. Alice defines the :ref:`SecretLockTransaction <secret-lock-transaction>` **TX1**:

* Mosaic: 10 alice token
* Recipient: Bob's address (Private Chain)
* Algorithm: h
* Secret:  h(proof)
* Duration: 96h
* Network: Private Chain

4. Alice **announces TX1 to the private network** and **shares with Bob the secret**.

5. Bob defines announces the following **SecretLockTransaction TX2** to the **public network**:

* Mosaic: 10 bob token
* Recipient: Alice's address (Public Chain)
* Algorithm: h
* Secret:  h(proof)
* Duration: 84h
* Network: Public Chain

.. note::  The amount of time in which funds can be unlocked should be a smaller time frame than TX1's. Alice knows the secret, so Bob must be sure he will have some time left after Alice releases the secret.

6. Alice announces the :ref:`SecretProofTransaction <secret-proof-transaction>` **TX3** to the **public network**. This transaction defines the encrypting algorithm used, the original proof and the secret.

7. Once TX3 is confirmed, the **proof** is revealed. **TX2 transaction is unlocked** and Alice receives the locked funds.

8. Bob picks the proof and announces the **SecretProofTransaction TX4** to the **private network**, receiving the locked funds from **TX1**.

******
Guides
******

.. postlist::
    :category: Cross-Chain Swaps
    :date: %A, %B %d, %Y
    :format: {title}
    :list-style: circle
    :excerpts:
    :sort:

*******
Schemas
*******

.. note:: Configuration parameters are :properties:`editable <config-network.properties>`. Public network configuration may differ.

.. _secret-lock-transaction:

SecretLockTransaction
=====================

Use a SecretLockTransaction to transfer mosaics between two accounts. The specified mosaics remain locked until a valid :ref:`SecretProofTransaction <secret-proof-transaction>` unlocks them.

If the transaction duration is reached without being proved, the locked amount goes back to the initiator of the SecretLockTransaction.

**Version**: 0x01

**Entity type**: 0x4152

**Inlines**:

* :ref:`Transaction <transaction>` or :ref:`EmbeddedTransaction <embedded-transaction>`

.. csv-table::
    :header: "Property", "Type", "Description"
    :delim: ;

    mosaic; :ref:`UnresolvedMosaic <unresolved-mosaic>`; Locked mosaic.
    duration; :schema:`BlockDuration <types.cats#L2>`; Number of blocks for which a lock should be valid. Duration is allowed to lie up to ``30`` days. If reached, the mosaics will be returned to the initiator.
    hashAlgorithm ; :ref:`LockHashAlgorithm<lock-hash-algorithm>`; Algorithm used to hash the proof.
    secret; :schema:`Hash256 <types.cats#L9>`; Proof hashed.
    recipientAddress; :schema:`UnresolvedAddress <types.cats#L7>`; Address that receives the funds once unlocked.

.. _secret-proof-transaction:

SecretProofTransaction
======================

Use a SecretProofTransaction to unlock :ref:`SecretLockTransactions <secret-lock-transaction>`.

The transaction must prove that it knows the *proof* that unlocks the mosaics.

**Version**: 0x01

**Entity type**: 0x4252

**Inlines**:

* :ref:`Transaction <transaction>` or :ref:`EmbeddedTransaction <embedded-transaction>`

.. csv-table::
    :header: "Property", "Type", "Description"
    :delim: ;

    hashAlgorithm ; :ref:`LockHashAlgorithm<lock-hash-algorithm>`; Algorithm used to hash the proof.
    secret; :schema:`Hash256 <types.cats#L9>`; Proof hashed.
    recipientAddress; :schema:`UnresolvedAddress <types.cats#L7>`; Address that receives the funds once unlocked.
    proofSize; uint16; Proof size in bytes.
    proof; array(byte, proofSize); Original random set of bytes.

.. _lock-hash-algorithm:

LockHashAlgorithm
=================

The list of supported hashing algorithms.

Enumeration: uint8

.. csv-table::
    :header: "Id", "Description"
    :delim: ;

    0 (Op_Sha3_256); Proof is hashed using SHA3-256.
    1 (Op_Keccak_256); Proof is hashed using Keccak (ETH compatibility).
    2 (Op_Hash_160); Proof is hashed twice: first with SHA-256 and then with RIPEMD-160 (bitcoin's OP_HASH160).
    3 (Op_Hash_256); Proof is hashed twice with SHA-256 (bitcoin's OP_HASH256).
