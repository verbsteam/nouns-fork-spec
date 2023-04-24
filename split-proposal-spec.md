# Split Proposal Spec

## High Level

- Introducing a "split proposal", a flow that lets Nouners activate a split as soon as the split threshold is reached.
- This feature always exists in one of two states: (1) escrow period and (2) split period.
- **Escrow period**:
  - Any owner can put their tokens in escrow, contributing towards reaching split threshold.
  - Token owners can pull their tokens out of escrow as long we haven't entered the split period.
  - We enter the split period when the number of escrowed tokens meets the split threshold, and someone calls the execute split DAO function.
  - Escrow does not allow voting & proposing; this works by the escrow not having a delegation feature, nor voting & proposing functions. The motivation of preventing voting is to add cost to going into escrow, to minimize 'parking' Nouns in escrow for long periods of time.
- **Split period**:
  - Starts upon the deployment of a New DAO, and goes for several days (e.g. 7 days), allowing more token owners to join New DAO.
  - During this period regular OG DAO proposals can't be executed, to prevent race conditions between the split flow and any malicious proposals.
  - Once a split is started it cannot be cancelled.
- New DAOs are deployed with vanilla ragequit in place; otherwise it's possible for a New DAO majority to collude to hurt a minority, and the minority wouldn't have any last resort if they can't reach the split threshold; furthermore bullies/attackers can recursively chase minorities into New DAOs in an undesired attrition war.

## Additional Details

### Split signaling

- When a Nouner escrows their Nouns or joins an active split, they may provide additional information on why they are splitting, to be recorded onchain:
  - Proposal Ids: one or more IDs of proposals that pushed them to split, whether by passing or by being shot down.
  - Reason: free text where they can elaborate, similar to how vote reasons work.

### What happens with OG Nouns that participate in a split?

- OG Nouns are sent to a new Nouns holding contract, controlled and owned by the OG DAO treasury.
- Nouns that are held there are excluded from the total supply used in key DAO calcuations: proposal threshold, quorum, split fair share and split threshold.
- Any Noun that is then transfered out of the new holding contract, goes back into the above calculations, and it's important to recognize this as a non-intuitive consequence.
- This is why Nouns are not simply held in the treasury; to make sure transfers go through a new function that helps Nouners understand the implication, e.g. by setting the function name to `transferNounsAndGrowTotalSupply` or something simialr, as well as emitting events that indicate the new (and greater) total supply used by the DAO.

## Design decisions summary

- Why DAO split over vanilla RQ?
  - So the event is rare and is not classified as distribution.
- Why do owners split and not delegates, and why can't we tie a split decision to a vote?
  - Nouns token delegation doesn't store which Nouns are delegated to each account, so we can't tell which Nouns participated in each vote.
  - This kind of action seems more appropriate for owners to take, as it involves transfering their Nouns
- Why allow vanilla ragequit in New DAOs, vs deploying New DAOs with the same DAO split as the OG DAO?
  - A split design would enable an attacker to follow splitters into their new DAO in a recursive attrition war (we have ideas on how to dilute the attacker, but they add undesired UX complexities).
  - A split design might lead to distributions in New DAO such that a majority there can bully a minority that doesn't have enough tokens to meet the split threshold (e.g. 140 Nouns split, 130 collude to bully the other 10).
- Why add the new split proposal flow?
  - If we tie splits to specific proposals, we would have to make proposal queuing time longer, which makes all honest proposals longer just for the rare case of needing to split; feels like a bad balance.
  - Tying to a proposal introduces complexities in case a proposal is canceled or takes longer due to objection period.
  - It's easy to support veto capture / holding hostage scenarios without being tied to a proposal.
  - An attacker might submit multiple malicious proposals at the same time. In such a scenario, a single split proposal is a natural shelling point for all Nouners to split on, whereas if we tie splits to proposals we might suffer from different voters attempting to split on different proposals and at the worst case not meeting the split threshold at all.
- Why add a 'no prop execution' constraint vs making sure the split proposal ends in time?
  - We're afraid of the timestamp approach leading to splits ending too early or too late; too early might mean not enough people are able to join, and too late can lead to a malicious proposal executing ahead of the split.
  - The timestamps approach might lead to people having to have two splits in parallel and moving from one to another
  - Overall UX feels risky
- Doesn't this design allow an attacker to split as well and grief Nouners into having to use the New DAO vanilla ragequit?
  - Yes, this griefing vector is a known issue of this design.
  - In an attempt to minimize the scope of this version of the split design, we're forced to work around the constraint that the Nouns token contract does not provide the information of which Noun ID was delegated to whom at a certain point in time (the vote snapshot block).
  - This constraint makes it impossible to condition a Noun's split on its vote.
  - Worth noting that hinging on votes and thus tying split to proposals, re-introduces the tricky attack vector of submitting multiple malicious proposals at once, to create confusion on which proposal everyone should split.
  - It's possible to iterate towards better solutions on all of these fronts, e.g. by updating the DAO to use NFT-based voting rather than voting balance and snapshots; we'd love to explore those possibilities in a different future version.

## An example story

- [Day 0] Evil puts up a malicious proposal to drain the treasury to an account they control.
- [Day 0] Alice immediately signals to split, sending her Nouns to an escrow contract; she also reaches out to all the Nouners she knows and asks them to signal split as well.
- [Day 1 - 6] Bob, Charlie and many other Nouners signal to split.
- [Day 6] Split threshold is reached, and Alice executes the split, moving the DAO into an active split period that will last 7 days.
- [Day 12] Evil tries to execute their proposal, but they can't because the split period is active.
- [Day 13] The last few Nouners still join the split.
- [Day 14] Evil executes the proposal, draining the OG DAO from whatever was left there.

Evil might choose to split into this New DAO with many of their tokens, in which case honest voters will be able to ragequit on their own before Evil is able to execute any malicious proposals. If Evil doesn't split with them, they may continue to run New DAO however they like.

## Griefing vector: bullying and splitting

Evil can leave the DAO after voting for a large spend, leaving them to foot the bill.

1. Evil submits a proposal that spends a lot, e.g. to donate half the treasury to Gitcoin.
2. The DAO votes to pass the proposal, both Evil and other voters.
3. Evil activates a split, leaving just enough Nouns behind to stay above proposal threshold.
4. The split happens, and anyone that didn't split is left with a much smaller treasury (they lost the Gitcoin donation + the funds from Evil splitting).

Such a scenario pushes all other Nouners into splitting as well, or to stay with a lower than expected treasury.

It's possible for honest actors to work together against this attack by:

1. Splitting with enough Nouns such that the bullying proposal fails due to insufficient funds, and stays this way until the proposal expires (e.g. they might pause auctions to prevent revenues or pass proposals to spend new revenues).
2. They can take the split Nouns that were sent to the treasury, and pass proposals that sell them back to honest accounts at the amount of ETH they have available from the split.
3. Honest actors can vanilla ragequit from New DAO, and use that ETH to buy back their old Nouns.
