# Nouns Fork Spec

## High Level

- Nouns Fork introduces a new forking user flow. This flow enables Nouners to start a fork as soon as the Fork Threshold is reached.
- The flow operates in one of two states: (1) Escrow Period and (2) Forking Period.
- **Escrow Period**:
  - Any Nouner can put their tokens in escrow, contributing towards reaching the fork threshold, which is a % of total supply.
  - Nouners can pull their tokens out of escrow as long as the forking period hasn’t started.
  - The forking period starts when the number of escrowed tokens meets the fork threshold, and someone calls the fork DAO function.
  - Tokens in escrow cannot be used for voting nor proposing. The motivation is to add cost to going into escrow, to minimize 'parking' Nouns in escrow for long periods of time.
- **Forking Period**:
  - Begins when the fork function is called; the fork function deploys a new Nouns DAO.
  - Lasts for several days (e.g. 7 days), allowing additional token holders to send their Nouns to the DAO and join the newly forked DAO.
  - During this period original DAO proposals can't be executed, to prevent race conditions between the forking flow and any malicious proposals.
  - Once forking starts it cannot be canceled.
- New DAOs are deployed with vanilla ragequit in place; otherwise it's possible for a new DAO majority to collude to hurt a minority, and the minority wouldn't have any last resort if they can't reach the forking threshold; furthermore bullies/attackers can recursively chase minorities into fork DAOs in an undesired attrition war.
- Funds (ETH & ERC20 tokens) are sent from the original DAO to the new DAO.
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
- Nouns that are held in escrow after the forking period starts or in the original DAO treasury are excluded from the total supply used in key DAO calculations: proposal threshold, quorum, fork funds calculation and fork threshold.
- Any Noun that is then transferred out of the escrow or treasury, goes back into the above calculations, and it's important to recognize this as a non-intuitive consequence.
- For this reason we're considering a change where Nouns won't be sent to the treasury, but rather to a holding contract; to make sure transfers go through a new function that helps Nouners understand the implication, e.g. by setting the function name to `transferNounsAndGrowTotalSupply` or something similar, as well as emitting events that indicate the new (and greater) total supply used by the DAO.

## What happens in the new DAO?

- All contracts are upgradable, so this is just the initial setup that can be changed through upgrade proposals.

### Governor

- There's delayed governance, meaning proposals can't be created until all escrow forkers have claimed their new tokens, or until the delayed governance expiration timestamp has passed. The intention is to protect slow claimers from abuse by fast claimers.
- There's vanilla ragequit, for reasons mentioned in the high level section.

### Token and Auction

- Token art is the same as the original Nouns, and forkers get to keep the same token IDs and same art per token.
- Auction house is paused and can be resumed via a proposal; it then starts selling at the same ID the original auction was when the fork happened.
- Founders reward remains active and sent to the Nounders account set in the original token.

## Example stories

### Griefing attack 1: force to quit the fork DAO

A majority attacker might choose to fork into this new DAO with many of their tokens, in which case minority token holders will be pushed to ragequit on their own before the attacker is able to execute any malicious proposals. If the attacker doesn't fork with them, they may continue to run their new DAO however they like.

### Griefing attack 2: bullying and splitting

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

- `escrowToFork(tokenIds)`
  - allows Nouners to signal their intent to fork.
  - transfers Nouns with the provided token IDs to the escrow contract.
  - contributes towards reaching the fork threshold.
- `withdrawFromForkEscrow(tokenIds)`
  - allows Nouners to pull their Nouns out of escrow.
  - transfers their Nouns back to them.
  - removes their contribution towards reaching the fork threshold.
- `fork()`
  - starts the forking period.
  - deploys a new Nouns DAO (see new DAO details below).
- `joinFork(tokenIds)`
  - allows Nouners to join the new DAO during the forking period, e.g. 7 days.
  - transfers their Nouns to the current DAO's treasury.
  - mints new DAO tokens to them.
- `withdrawNounsFromEscrow(tokenIds)`
  - allows the DAO to withdraw Nouns from the escrow contract.
  - only works once the forking period starts.
- `adjustedTotalSupply`
  - returns the total supply of Nouns that are not in escrow or in the treasury.

New admin functions (can only be executed via proposals):

- `_setForkThreshold(thresholdBPs)`
  - sets the minimum % out of Nouns total supply that need to be escrowed to start a forking period.
- `_setForkPeriod(period)`
  - sets the duration of the forking period.
- `_setForkEscrow(escrowAddress)`
  - sets the escrow contract address.
- `_setForkDAODeployer(deployerAddress)`
  - sets the address of the contract that deploys fork DAOs.
- `_setErc20TokensToIncludeInFork(erc20Addresses)`
  - sets the list of ERC20 tokens that will be transferred to fork DAOs.

Modified functions:

- `execute(proposalId)`
  - reverts during an active forking period.
- `proposalThreshold()`
  - uses `adjustedTotalSupply` instead of `nouns.totalSupply`.
- `quorumVotes(proposalId)`
  - uses `adjustedTotalSupply` instead of `nouns.totalSupply`.
- `propose(txs)`
  - uses `adjustedTotalSupply` instead of `nouns.totalSupply` when setting totalSupply on a new proposal.
- `proposeOnTimelockV1(txs)`
  - enables proposals that get executed on the original Timelock contract.

### Timelock (NounsDAOExecutorV2)

The current Timelock contract is not upgradable, so we need to deploy a new one to add the new functions that support forking fund transfers. The migration plan to the new Timelock is detailed further in a section below.

TLDR:

- Based on V1.
- Adds upgradability.
- Adds functions to send assets to fork DAOs.

New functions:

- `sendETH(to, amount)`
  - sends ETH to the provided address, which would be the fork DAO's treasury.
  - can only be called by the DAO governor contract.
- `sendERC20(to, amount)`
  - sends ERC20 tokens to the provided address, which would be the fork DAO's treasury.
  - can only be called by the DAO governor contract.

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
- `closeEscrow()`
  - is called by the DAO when the forking period starts.
  - increments the latest fork ID.
  - allows the DAO to withdraw Nouns from escrow.
  - can only be called by the original DAO governor contract.

### ForkDAODeployer

- `deploy()`
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

The same as Timelock V2 in the original DAO, upgradable and with the new functions to send funds.

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
  - only the OG DAO is allowed to call this.
  - can only be called during the forking period.

### Auction

TLDR:

- Upgradability changed from the old transparent proxy design to the new UUPS pattern.
- Everything else is the same.

## Timelock Migration

1. TimelockV2 is deployed, with the DAO as its admin.
2. A DAO proposal that:
   1. Upgrades to DAOv3 pointing to the new timelock.
   2. Transfers all ETH to the new timelock.
   3. Transfers all ERC20s to the new timelock.
      - Some stETH dust will be left behind. We can decide to write a contract which transfers the exact amount.
   4. Changes nouns.eth to new timelock.
   5. Transfer ownership of TokenBuyer & Payer to new timelock.
3. Maybe later:
   1. Approve new timelock the transfer NFTs in old timelock, e.g. lilnouns
   2. Change the address set to receive founder rewards from other nounish projects.
