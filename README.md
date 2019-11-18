# Bisq

Bisq is a decentralized trading platform that provides a censorship resistent, decentralized way to buy / sell BTC with fiat (as well as altcoins).
One challange of such a system is ensuring traders can trust each other to execute trades by posting some sort of collateral. Additionally there must be a process for reconciliation should a trade go badly for technical or nefarious reasons.

The current trade protocol uses BTC stored in a multisig address for this purpose, however there are some drawbacks to this which include:
- The number and size of transactions required to setup and execute a trade is quite high which results in a lot of overhead being paid as transaction fees
- In order to confiscate and reconsile funds in a trade gone bad (via mediation / arbitration) a pre-signed transaction is created that lets a trader send the funds to the bisq donation address (after a certain point in time). The donation address thus holder becomes an implicit participant of all trades.

With the launch of the Bisq DAO and the associated BSQ token some new options present themselves.
Because BSQ can be confiscated and distributed by the DAO via voting it lends itself to being used as collateral.
This would remove the need for the donation address transaction and also reduce the number and size of transactions over-all, given that a locked up bond can be reused for multiple trades.

## Using bonded BSQ as collateral for trading

In order to use BSQ as collateral for any given trade the following must be ensured:
- A bond (or slice thereof) can be specified to be the collateral for a given trade
- A bond (or slice thereof) cannot be associated with more than 1 trade at a time
- A trader must be able to proove to the counterparty that a trade is backed by a bond
- A trader must be able to proove that the bond is not allocated to any other purpose
- To protect privacy a trader should not need to reveal any specifics about any other trade other than the one at hand to prove that the trade is collateralized
- The collateral can only be re-clamied for re-use once both parties agree on an outcome of a trade

Uniquely commiting a bond (or a slice thereof) to act as collateral for a trade is equivilent to solving the double-spend problem.
Previous suggestions on how to do bonded trading have proposed traders should use the p2p flood all gossip protocol to announce the collateral commitments and rely on attestations from 3rd parties that witnessed the announcement for verification.
This is equivilent to accepting 0-conf transactions that have been anounced to the bitcoin mem-pool as final without them having been confirmed in a block.
Due to the nature of the gossip protocol and the byzantine generals problem it is impossible to rule out issues arrising from double allocation (either due to bugs or as a deliberate scam) when relying only on the gossip layer.

To get around this issue a solid proof-of-publication mechanism must be defined.
The medium in which the announcement should take place must itself be decentralized and censorship resistant or it would defeat the purpose of the platform.
Using a bitcoin transaction to announce a commitment of a certain collateral to a certain trade would be a potential starting point, but since the goal is to reduce the number and size of transactions required on the bitcoin blockchain a 1-to-1 relationship between trades and announcements is not a very attractive solution.
For this approach to be considered we must be able to significantly scale the number of announcements per-transaction.

## Using Single-Use seals to commit collateral on per-bond ledgers

This document proposes to use the semantics of the [single-use seals](https://petertodd.org/2017/scalable-single-use-seal-asset-transfer) concept put forward by Peter Todd to construct per-bond mini-ledgers that document the allocation of bond-slices to trades.
The state of the individual ledgers is finalized via 'anchor' transactions on bitcoin.
Each anchor transaction can be used to commit to the state of 1 or multiple ledgers each ledger in turn containing multiple commitments (due to bonds being 'sliced' into different chunks for collateral).
The anchor transactions can be initiated by a trader as needed when a final commitment is required and can be batched with other transaction content (eg. paying BTC to a counterparty) to further improve scaling.
Alternatively the bisq network could also choose to provide a service that would batch anchor transactions from multiple parties for additional scaling benefits, though this would trade off some privacy and cencorship resistance.

## Anchor transactions

The per-bond ledger is organised into blocks and finalized by anchor transactions.
The anchor transaction must spend the UTXO previously determined to be the Anchor Continuation Output.
The anchor transaction has an OP_RETURN that commits to the next Anchor Continuation Output as well as the merkle root of the finalized block:
OP_RETURN = Hash(txout|merkelroot)
The Anchor continuation output is initially the BTC change output of the bond TX and subsequently identified via OP_RETURN

## per bond ledger

The per-bond ledger records state transitions of bond-slices via storing witnesses to operations in a [sparse merkle tree](https://eprint.iacr.org/2016/683.pdf).
This allows for efficient proof-of-existence as well as proof-of-non-existance (important for showing that a slice hasn't been double committed).

The initial genesis state of a bond is:
CommittableSlice(amount, p)
where amount = the total bond amount and p = the pub key of the initial Anchor Continuation Output

The witnesses are computed as follows:
witness = Sign(CurrentState, NextState, p)
where p is the pub-key associated with CurrentState

Operations that can be performed are:
- Commit: associate 1 - n slices with a given trade
- Reclaim: once a trade has completed or the lock time has run out
- Split: Split up a CommitableSlice into n sub-slices (for committing to multiple trades).

When inserted into a block p 

Split(Input, Outputs)
where
Input = CommittableSlice(amount, pin)
Output[n] = CommittableSlice(amountOutN, p[n])
Sum amountOut[0-n] <= Input.amount

The per-bond ledger determins the state and history of operations.
It starts with a genesis state of
CommittableSlice(amount, p)
where amount = the total bond amount and p = the public key of the 

States: spendable, committed
Operations: split, commit, reclaim

Commit operation:
for each input Sign(Inputs, ChangeOutput, reclaim-hash, p) => w
where Input = CommittableSlice(amount, p)
ChangeOutput = CommittableSlice(amount, b)
Committed value = Sum(Inputs.amount) - ChangeOutput.amount
ChangeOutput.amount < Sum(Inputs.amount) - committed value
reclaim-hash = Hash(counterparty-secret)

Reclaim Operation:
Sign(CommittableSlice(amount, p), reclaim-secret) => w
where Hash(reclaim-secret) = reclaim-hash
CommittableSlice.amount <= committed value

To execute a trade:
M + T exchange reclaim-hash
M + T create Commit Seal and Anchor it
M + T proove to each other the validity of their commitment
Exchange assets
Exchange reclaim-preimages
