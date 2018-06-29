```
shortname: [18/TEP]
name: Transactional Election Process
type: standard
status: raw
editor: Alberto Granzotto <alberto@bigchaindb.com>
```


# Transactional Election Process

## Abstract
<!-- The abstract is a short (~200 word) description of the technical issue being addressed. -->
This specification introduces the new concept of **Election**. An Election is an asynchronous process that when successful triggers Network wide changes synchronously (i.e. at the same *block height*). An Election is started by any Validator in the Network, called **Initiatior**. The Election itself and all casted Votes are transactions, hence stored in the blockchain. This enables new Validators to replay Network changes incrementally, while syncing.

## Motivation
<!--The motivation is critical for BEPs that want to change the BigchainDB protocol. It should clearly explain why the existing protocol BEP is inadequate to address the problem that the BEP solves. BEP submissions without sufficient motivation may be rejected outright.-->
Changing the shape of a BigchainDB Network at runtime is an important requirement. While this BEP does not address this issue directly, it wants to solve the limitations we had with [Upsert Validators (BEP-3)][BEP-3] by giving a tool that can be used this and other situations. BEP-3 addresses a really interesting use case: adding a Member to a running Network. Since changes in the Validator Set are not themselves expressed as transactions but are included into the metadata of an arbitrary block, all the nodes have to add a validator simultaneously—if less than the majority of the nodes attempt to include a Validator, the block fails and the whole Network can stuck. Since those changes are **not** stored in the blockchain itself, a new Node that needs to sync with the Network will fail in validating blocks from a specific *height* on. The *height* will match the one of the first block voted by a new Validator that was **not** included in the `genesis.json` file (and Tendermint will fail with `Invalid commit (last validator set size) vs (number of precommits)`).

The abstract version of the problem is "make Network wide decisions". We need a way for Members to vote on proposals, and implement them as soon as the quorum of ⅔ is reached. In this BEP, a proposal is named *Election*, a vote is a fungible token formalized with the capitalized name *Vote*. Elections and Votes are stored in the blockchain, so a new Node that syncs with the Network will be able to replay them locally, and reach the same state of all other Members.

The basic idea is to formalize the concept of an Election storing its data in a [BigchainDB Transaction Spec v2][BEP-13].
By storing it in a BigchainDB Network, it allows Members to vote asynchronously, and Validators to apply changes synchronously.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations. It MAY describe the impact on data models, API endpoints, security, performance, end users, deployment, documentation, and testing.-->

Let the validator set be denoted by `Vₜ` at time `t`, a Member of a BigchainDB Network can start a new Election. This member is called **Initiator** (`vₜᵢ` s.t. `vₜᵢ ∊ Vₜ`) .

### Valid Election

An Election proposal is a transaction representing the matter of change. The _Initiator_ `vₜᵢ`, issues a create election transaction (referred as `ECₜ`). The election MUST contain outputs  such that each output is populated with the public key of the validator `vₖ` such that `vₖ ∊ Vₜ`, and an amount equal to the current power `P(vₖ, t)` of the validator.


### Voting

Once `ECₜ` is committed in a block, the election starts. Independently and asynchronously, each validator may spend its vote tokens (referred as `Tₖ`) to the election address (`ECₜₐ`) to show agreement on the matter of change. The Election Address `ECₜₐ` is the `id` of the transaction `ECₜ`.

NOTE: Once the vote tokens `Tₖ` have been transferred to the election address `ECₐ` it is not possible to transfer it again, because the private key is not known.


### Concluding Election

At time `tn` let,
- Validator set be denoted by `Vₜₙ` s.t. `vᵢ ∊ Vₜₙ` 
- `vₘ ∊ Vₘ`, where `Vₘ` denotes the set of public keys who voted for election `ECₜ`
- `Tₙₑᵤ` be the newly received vote token


#### Constrained approach (Approach 1)
In the constrained approach any change to the validator set invalidates all the elections which were initiated with a different validator set.

If below conditions hold true then the election is concluded and the proposed change is applied,
1. `Vₜₙ = Vₜ` 
2. `Tₙₑᵤ + ∑ₖ Tₖ > (2/3) ∑ₖ P(vₖ, t)` where `vₖ ∊ Vₜ` and `Tₖ` denotes the vote tokens received at `ECₐ` prior the the current token
3. `∑ₖ Tₖ < (2/3) ∑ₖ P(vₖ, t)`


#### Generalized approach (Approach 2)
The generalized constraints can tolerate a certain degree of change to the validator set.

If below conditions hold then the election is concluded and the proposed change is applied,
- `Tₙₑᵤ + ∑ₖ Tₖ > (2/3) ∑ₖ P(vₖ, t)` where `vₖ ∊ Vₜ` and `Tₖ` denotes the vote tokens received at `ECₐ` prior the the current token
- `∑ₘ P(vₘ, tn) > (2/3) ∑ᵢ P(vᵢ, tn)`
- `∑ₖ Tₖ < (2/3) ∑ₖ P(vₖ, t)` 

The above constraints state that if the validators with which an election `ECₜ` was initiated still hold super-majority then the election can be concluded.


### Applying change

During the `end_block` call, all transactions about to be committed are checked. Every vote token triggers a function (which implements **Approch 1** or **Approach 2**) that checks the necessary conditions. If the function returns `True` then the current validator applies the suggested change in `ECₜ`. Given the BFT nature of the system, all non-Byzantine Validator will commit the change at the same block height.


## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->
Another idea to implement this protocol is to use a multi signature transaction.

## Backwards Compatibility
<!--All BEPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The BEP must explain how the author proposes to deal with these incompatibilities. BEP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
This BEP is fully backwards compatible.

## Implementation
<!--The implementations must be completed before any BEP is given status "stable", but it need not be completed before the BEP is accepted. While there is merit to the approach of reaching consensus on the BEP and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->
This section will be filled in once there *is* an implementation.

# Copyright Waiver
To the extent possible under law, the person who associated CC0 with this work has waived all copyright and related or neighboring rights to this work.

[BEP-3]: ../3
[BEP-13]: ../13
