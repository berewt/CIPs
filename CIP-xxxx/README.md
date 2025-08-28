---
CIP: xxxx
Title: Verifiable Peras Certificates
Category: Ledger
Status: Proposed
Authors:
    - Nicolas Biri <nicolas.biri@iohk.io>
Implementors: []
Discussions:
    - https://github.com/cardano-foundation/CIPs/pull/?
Created: 2025-08-22
License: CC-BY-4.0
---

## Abstract

We propose to include within the block header of the first block of each epoch
a commitment proof of the stake distribution of this epoch and of the Peras
voting public keys of the SPOs.
Doing so, light client would have a way to verify Peras certificates and to take
advantage of the high confidence in the fork they provide.
Combined with a
[commitment to the set of transactions in the block header][MerkleTxs],
it gives us enough information to verify transaction inclusion in a trustless
setup.

## Motivation: why is this CIP necessary?

The complexity of building trustless state proof of Cardano is well documented
(see [CPS-0019?](https://github.com/cardano-foundation/CIPs/pull/942)).
The key challenges boil down to two major questions:
- How can we easily claim that a transaction is included in a block?
  (which is addressed in the draft CIP about
  [Merkle root of transactions in the block header][MerkleTxs])
- How can we get high confidence that the fork we observe is the longest one?
  (which is addressed in this CIP)

[Peras][Peras] introduces a way to obtain high-confidence in a fork of the chain
through the introduction of certificates that enough SPOs agree on this fork.
Doing so, Peras improves the settlement time of Cardano, as a transaction that
is covered by a few votes certificates has a very low risk of being rolled back.

Unfortunately, Peras certificates has three major drawbacks:

1. They aren't published on-chain.
   One could easily provide the certificate off chain, but we would need to
   verify the certificate.
2. The public keys of the signers aren't commited on chain either,
   which makes a forged certificate indistinguishable from a valid one if we
   don't run a full node.
3. This verification relies on stake distribution, which isn't published
   on-chain.

To address this problem, we propose to include two commitments proof in
distribution in the block header of the first block of each epoch.
One is a commitment proof of the stake distribution of the previous epoch,
and the other a commitment proof of the Peras voting public keys of the SPOs.
If we add a way to get off-chain the stake distribution of the previous epoch
and the full list of public keys used to sign the Peras certificates,
we can verify the Peras certificates against these commitments.


More generally, knowing about the stake distribution is helpful to verify any
type of consensus activity within the network.
Many parachains solutions (L2, partner chains, etc.) rely on the stake of the
L1.
Being able to verify this distribution can also lead to simplification in their
design.


## Specification

On the first block of each epoch, we add within the block header the root of
two Merkle trees representing respectively the stake pool distribution
of the last epoch, and the public keys used to sign Peras certificates.
The Merkle tree for the stake pool distribution is a binary tree built from a
list of pairs `(pool_id, stake)`.
The Merkle tree for the Peras public keys is a binary tree built from a list of
pairs `(pool_id, public_key)`.

**TODO:** Consider whether we want to move to Mekle Patricia forest.

**TODO:** Define the change in the block header structure.

We then suggest to update the block header structure, adding two fields
`bheaderStakeDistRoot` and `bheaderPerasPubKeysRoot` to the `BHBody` type
(see below).

```haskell
data BHBody c = BHBody
  { bheaderBlockNo :: !BlockNo
  -- ^ block number
  , bheaderSlotNo :: !SlotNo
  -- ^ block slot
  , bheaderPrev :: !PrevHash
  -- ^ Hash of the previous block header
  , bheaderVk :: !(VKey 'BlockIssuer)
  -- ^ verification key of block issuer
  , bheaderVrfVk :: !(VRF.VerKeyVRF (VRF c))
  -- ^ VRF verification key for block issuer
  , bheaderEta :: !(VRF.CertifiedVRF (VRF c) Nonce)
  -- ^ block nonce
  , bheaderL :: !(VRF.CertifiedVRF (VRF c) Natural)
  -- ^ leader election value
  , bsize :: !Word32
  -- ^ Size of the block body
  , bhash :: !(Hash HASH EraIndependentBlockBody)
  -- ^ Hash of block body
  , bheaderOCert :: !(OCert c)
  -- ^ operational certificate
  , bheaderStakeDistRoot :: !(Maybe (Hash HASH StakeDist))
  -- ^ Merkle root of the stake pool distribution of the previous epoch
  , bheaderPerasPubKeysRoot :: !(Maybe (Hash HASH PerasPubKeys))
  -- ^ Merkle root of the Peras voting public keys of the SPOs
  , bprotver :: !ProtVer
  -- ^ protocol version
  }
```



## Rationale: how does this CIP achieve its goals?

The main goal of this CIP, once Peras is implemented, is to allow the
construction of trustless state proofs of the Cardano chain.
Combined with integration of a
[commitment of the set of transactions in the block header][MerkleTxs],
it allows trustless state proof of inclusion of a transaction in the Cardano
chain.

With the two proposed commitments, if we have access off-chain to the stake pool
distribution, and if we agreed on a initial block (which we
can do easily by using at worst the first block of the chain, or by agreeing on
a immutable block), we can build a trustless state proof that a transaction is
covered by an arbitrary number of Peras certificates, which gives us a way to
arbitrage between settlement time and settlement confidence.

While different types of proofs can be built, we propose here a naive approach
to build a proof that a transaction is covered by Peras certificates.
Such a proof can use, as instances:

- The hash of the initial block, which is the first block of the chain or a block agreed
  upon by the parties.
- The number of Peras certificates that we want to have on top of the
  transaction.
- The transaction id.

And as witnesses:
- A chain of blocks from the initial block to the one corresponding to the one
  corresponding to the last Peras certificate.
- The stake pool distribution of the previous epoch.
- The public keys of the SPOs used to sign the Peras certificates.
- The peras certificates themselves.

The circuit of the proof should verify that:
- The transaction is included in the set of transactions of one of the block
  from the initial block.
- This block is covered by the given Peras certificates.
- The Peras certificates are valid, i.e. the public keys used to sign them
  are in the list of public keys of the SPOs and they have enough stake.

Obviously, we can add logic on top of that and verify specific part of the
transaction if needed.

## Path to Active

### Acceptance Criteria

- [ ] Both commitment proofs are included in the block header of the first block of each epoch.

### Implementation Plan

- [ ] Peras is implemented and deployed.
- [ ] Agreement by the Ledger team as defined in CIP-0084 under Expectations for
ledger CIPs.
- [ ] The implementation of the CIP is prioritized for a chosen hard fork.


<!-- OPTIONAL SECTIONS: see CIP-0001 > Document > Structure table -->

## Copyright

This CIP is licensed under [Apache-2.0](http://www.apache.org/licenses/LICENSE-2.0).

[Peras]: https://github.com/cardano-foundation/CIPs/blob/master/CIP-0140
[MerkleTxs]: https://github.com/cardano-foundation/CIPs/pull/964
