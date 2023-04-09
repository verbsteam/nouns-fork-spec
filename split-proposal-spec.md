# Split Proposal Spec

## High Level

- Introducing a "split proposal", a flow that lets Nouners activate a split as soon as the split threshold is reached.
- This feature always exists in one of two states: (1) escrow period and (2) split period.
- **Escrow period**:
  - Any owner can put their tokens in escrow, contributing towards reaching split threshold.
  - Token owners can pull their tokens out of escrow as long we haven't entered the split period.
  - We enter the split period when the number of escrowed tokens meets the split threshold, and someone calls the execute split DAO function.
- **Split period**:
  - Starts upon the deployment of a New DAO, and goes for several days (e.g. 7 days), allowing more token owners to join New DAO.
  - During this period regular OG DAO proposals can't be executed, to prevent race conditions between the split flow and any malicious proposals.
  - Once a split is started it cannot be cancelled.
- New DAOs are deployed with vanilla ragequit in place; otherwise it's possible for a New DAO majority to collude to hurt a minority, and the minority wouldn't have any last resort if they can't reach the split threshold; furthermore bullies/attackers can recursively chase minorities into New DAOs in an undesired attrition war.

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
- Why add a 'no prop execution' constraint vs making sure the split proposal ends in time?
  - We're afraid of the timestamp approach leading to splits ending too early or too late; too early might mean not enough people are able to join, and too late can lead to a malicious proposal executing ahead of the split.
  - The timestamps approach might lead to people having to have two splits in parallel and moving from one to another
  - Overall UX feels risky

## An example story

- [Day 0] Evil puts up a malicious proposal to drain the treasury to an account they control.
- [Day 0] Alice immediately signals to split, sending her Nouns to an escrow contract; she also reaches out to all the Nouners she knows and asks them to signal split as well.
- [Day 1 - 6] Bob, Charlie and many other Nouners signal to split.
- [Day 6] Split threshold is reached, and Alice executes the split, moving the DAO into an active split period that will last 7 days.
- [Day 12] Evil tries to execute their proposal, but they can't because the split period is active.
- [Day 13] The last few Nouners still join the split.
- [Day 14] Evil executes the proposal, draining the OG DAO from whatever was left there.

Evil might choose to split into this New DAO with many of their tokens, in which case honest voters will be able to ragequit on their own before Evil is able to execute any malicious proposals. If Evil doesn't split with them, they may continue to run New DAO however they like.
