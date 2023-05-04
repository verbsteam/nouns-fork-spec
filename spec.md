# Nouns Fork Spec

## High Level

- Nouns Fork introduces a new forking user flow. This flow enables Nouners to start a fork as soon as the Fork Threshold is reached.
- The feature operates in one of two states: (1) Escrow Period and (2) Forking Period.
- **Escrow Period**:
  - Any Nouner can put their tokens in escrow, contributing towards reaching fork threshold.
  - Nouners can pull their tokens out of escrow as long we haven't entered the forking period.
  - We enter the forking period when the number of escrowed tokens meets the fork threshold, and someone calls the fork DAO function.
  - Escrow does not allow voting & proposing; this works by the escrow not having a delegation feature, nor voting & proposing functions. The motivation of preventing voting is to add cost to going into escrow, to minimize 'parking' Nouns in escrow for long periods of time.
- **Forking Period**:
  - Begins when the fork function is called, and a new Nouns DAO is deployed; goes for several days (e.g. 7 days), allowing more token owners to join the new DAO.
  - During this period original DAO proposals can't be executed, to prevent race conditions between the forking flow and any malicious proposals.
  - Once forking starts it cannot be cancelled.
- New DAOs are deployed with vanilla ragequit in place; otherwise it's possible for a new DAO majority to collude to hurt a minority, and the minority wouldn't have any last resort if they can't reach the forking threshold; furthermore bullies/attackers can recursively chase minorities into fork DAOs in an undesired attrition war.
- Funds are sent from the original DAO to the new DAO.
- Forking Nouners claim new DAO tokens with the same IDs and same art as the Nouns they returned to the original DAO.

## Additional Details

### Fork signaling

- When a Nouner escrows their Nouns or joins an active forking period, they may provide additional information on why they are splitting, to be recorded onchain:
  - Proposal Ids: one or more IDs of proposals that pushed them to fork, whether by passing or by being shot down.
  - Reason: free text where they can elaborate, similar to how vote reasons work.

### What happens with original Nouns that participate in a fork?

- During the escrow period Nouns are held in the escrow contract.
- During the forking period additional forking Nouns are sent directly to the original DAO's treasury.
- Once the forking period is over, all Nouns that are in the escrow contract can be withdrawn by the original DAO via a proposal.
- Nouns that are held in escrow or in the original DAO treasury are excluded from the total supply used in key DAO calcuations: proposal threshold, quorum, split fair share and split threshold.
- Any Noun that is then transfered out of the new holding contract, goes back into the above calculations, and it's important to recognize this as a non-intuitive consequence.
- This is why Nouns are not simply held in the treasury; to make sure transfers go through a new function that helps Nouners understand the implication, e.g. by setting the function name to `transferNounsAndGrowTotalSupply` or something simialr, as well as emitting events that indicate the new (and greater) total supply used by the DAO.

## Example stories

### A malicious proposal

- [Day 0] Evil puts up a malicious proposal to drain the treasury to an account they control.
- [Day 0] Alice immediately signals to fork, sending her Nouns to an escrow contract; she also reaches out to all the Nouners she knows and asks them to signal fork as well.
- [Day 1 - 6] Bob, Charlie and many other Nouners signal to fork.
- [Day 6] Fork threshold is reached, and Alice executes the fork, moving the DAO into an active forking period that will last 7 days.
- [Day 12] Evil tries to execute their proposal, but they can't because the forking period is active.
- [Day 13] The last few Nouners still join the fork.
- [Day 14] Evil executes the proposal, draining the OG DAO from whatever was left there.

Evil might choose to fork into this new DAO with many of their tokens, in which case honest token holders will be able to ragequit on their own before Evil is able to execute any malicious proposals. If Evil doesn't fork with them, they may continue to run their new DAO however they like.

### Griefing vector: bullying and splitting

Evil can leave the DAO after voting for a large spend, leaving them to foot the bill.

1. Evil submits a proposal that spends a lot, e.g. to donate half the treasury to Gitcoin.
2. The DAO votes to pass the proposal, both Evil and other voters.
3. Evil activates a fork, leaving just enough Nouns behind to stay above proposal threshold.
4. The fork happens, and anyone that didn't leave is left with a much smaller treasury (they lost the Gitcoin donation + the funds from Evil splitting).

Such a scenario pushes all other Nouners into forking as well, or to stay with a lower than expected treasury.

It's possible for honest actors to work together against this attack by:

1. Forking with enough Nouns such that the bullying proposal fails due to insufficient funds, and stays this way until the proposal expires (e.g. they might pause auctions to prevent revenues or pass proposals to spend new revenues).
2. They can take the fork Nouns that were sent to the treasury, and pass proposals that sell them back to honest accounts at the amount of ETH they have available from forking.
3. Honest actors can vanilla ragequit from the new DAO, and use that ETH to buy back their old Nouns.

## Original DAO Contracts

### Governor (NounsDAOLogicV3)

New functions:

- `signalFork(tokenIds)`
  - allows Nouners to signal their intent to fork.
  - transfers Nouns with the provided token IDs to the escrow contract.
  - contributes towards reaching the fork threshold.
- `unsignalFork(tokenIds)`
  - allows Nouners to pull their Nouns out of escrow.
  - transfers their Nouns back to them.
  - removes their contribution towards reaching the fork threshold.
- `fork()`
  - starts the forking period.
  - deploys a new Nouns DAO (see new DAO details below).
- `joinFork(tokenIds)`
  - allows Nouners to join the new DAO.
  - transfers their Nouns to the current DAO's treasury.
  - mints new DAO tokens to them.
- `withdrawNounsFromEscrow(tokenIds)`
  - allows the DAO to withdraw Nouns from the escrow contract.
  - only works once the forking period starts.
- `adjustedTotalSupply`
  - returns the total supply of Nouns that are not in escrow or in the treasury.

New admin functions (can only be executed via proposals):

- `_setSplitEscrow(escrowAddress)`
  - sets the escrow contract address.
- `_setSplitDAODeployer(deployerAddress)`
  - sets the address of the contract that deploys fork DAOs.
- `_setErc20TokensToIncludeInSplit(erc20Addresses)`
  - sets the list of ERC20 tokens that will be transferred to fork DAOs.

### Timelock (NounsDAOExecutorV2)

TLDR:

- Based on V1.
- Adds upgradability.
- Adds functions to send assets to fork DAOs.

New functions:

- `sendETH(to, amount)`
  - sends ETH to the provided address, which would be the fork DAO's treasury.
- `sendERC20(to, amount)`
  - sends ERC20 tokens to the provided address, which would be the fork DAO's treasury.

### NounsDAOForkEscrow

- `markOwner(owner, tokenIds)`
  - documents owner's sending tokenIds to escrow, and the specific fork ID they are participating in.
  - increments the fork's escrowed Nouns count.
  - can only be called by the original DAO governor contract.
- `returnTokensToOwner(owner, tokenIds)`
  - sends Nouns back to their owner.
  - decrements the fork's escrowed Nouns count.
  - can only be called before the forking period starts.
  - can only be called by the original DAO governor contract.
- `withdrawTokensToDAO(tokenIds, to)`
  - sends Nouns to the original DAO's treasury.
  - can only be called by the original DAO's treasury (timelock) contract.
- `closeEscrow`
  - is called by the DAO when the forking period starts.
  - increments the latest fork ID.
  - allows the DAO to withdraw Nouns from escrow.
  - can only be called by the original DAO governor contract.

### ForkDAODeployer

- `deploy`
  - deploys a new Nouns DAO, including a token, auction, governor and treasury.

## Fork DAO Contracts

### Governor

TLDR:

- Based on V1 (NounsDAOLogicV1).
- Adds delayed governance.
- Adds vanilla ragequit.
- Upgradability changed from the old proxy design to the new UUPS pattern.

New functions:

- `quit(tokenIds)`
  - allows Nouners to quit the DAO on their own.
  - transfers their tokens to the DAO's treasury.

Modified functions:

- `propose(proposal txs)`
  - Blocks proposals until all claimable tokens have been claimed, or until the governance delay expiration has passed.

New admin functions (can only be executed via proposals):

- `_setErc20TokensToIncludeInSplit(erc20Addresses)`
  - sets the list of ERC20 tokens that will be transferred to fork DAOs.

### Timelock

The same as the new timelock in the original DAO.

### Token

TLDR:

- Made upgradable (fork DAOs can upgrade it such that it's no longer upgradable).
- Supports minting / claiming tokens with the same ID and art as OG DAO.

New functions:

- `claimFromEscrow(tokenIds)`
  - allows Nouners to claim new tokens with the same ID and art as their OG Nouns in escrow.
  - mints new DAO tokens to them.
  - decrements the counter of remaining tokens to claim.
- `claimDuringForkingPeriod(tokenIds)`
  - allows Nouners to mint new tokens with the same ID and art as their OG Nouns.

### Auction

TLDR:

- Upgradability changed from the old transparent proxy design to the new UUPS pattern.
- Everything else is the same.
