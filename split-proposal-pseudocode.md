# Split Proposal

## High level spec

- any Nouner can create a new split proposal, as long as there isn't already an active one.
- the proposal to split is in 'escrow period' from creation until threshold is reached, or until all Nouns pull out of escrow.
- once split threshold is reached anyone can execute the split, moving the DAO into a 'split period', during which Nouners can split with more nouns immediately into the New DAO, without parking in escrow first; the split period duration is a DAO-configured variable, e.g. set to 7 days.
- during the 'split period' OG DAO proposals cannot be executed.
- once a split proposal enters the split period, the split cannot be cancelled, and escrowed Nouns are committed to the split and cannot be pulled out.
- New DAO support 'vanilla ragequit', i.e. any Nouner can immediately quit any time with their fair share; this design protects Nouners against a malcious actor or any majority bullying in New DAO.

## Design decisions summary

- why DAO split over vanilla RQ?
  - so the event is rare and is not classified as distribution
- why owners split and not delegates, and why can't we tie a split decision to a vote?
  - Nouns token delegation doesn't store which Nouns are delegated to each account, so we can't tell which Nouns participated in each vote
  - This kind of action seems more appropriate for owners to take, as it involves transfering their Nouns
- why allow vanilla ragequit in New DAOs, vs deploying New DAOs with the same DAO split as the OG DAO?
  - a split design would enable an attacker to follow splitters into their new DAO in a recursive attrition war (we have ideas on how to dilute the attacker, but they add undesired UX complexities)
  - a split design might lead to distributions in New DAO such that a majority there can bully a minority that doesn't have enough tokens to meet the split threshold (e.g. 140 Nouns split, 130 collude to bully the other 10)
- why add the new split proposal flow?
  - if we tie splits to specific proposals, we would have to make proposal queuing time longer, which makes all honest proposals longer just for the rare case of needing to split; feels like a bad balance
- why add a 'no prop execution' constraint vs making sure the split proposal ends in time?
  - we're afraid of the timestamp approach leading to splits ending too early or too late; too early might mean not enough people are able to join, and too late can lead to a malicious proposal executing ahead of the split
  - the timestamps approach might lead to people having to have two splits in parallel and moving from one to another
  - overall UX feels risky

## DAO

```jsx
// execute proposal same as today, only it blocks execution during a split period
function execute(proposalId):
    require !splitPeriodActive()
    // current execute logic follows below

// puts tokens in escrow
function signalSplit(tokenIds):
    // once split is active we don't allow escrows; instead use joinSplit
    require !splitPeriodActive()

    escrow = getOrCreateEscrow(owner: msg.sender, splitProposalId)
    nouns.transferFrom(msg.sender, escrow, tokenIds)
    escrowCount[splitProposalId] += tokenIds.length

    if (!escrows[splitProposalId][escrow]):
        escrows[splitProposalId][escrow] = true
        escrowArrays[splitProposalId].push(escrow)

// takes tokens out of escrow
function unsignalSplit(tokenIds, to):
    require !splitPeriodActive()

    escrow = getEscrow(owner: msg.sender, splitProposalId)
    escrow.returnToOwner(tokenIds)
    escrowCount[splitProposalId] -= tokenIds.length


// creates New DAO, sends it money, and kicks off 'split period' when more Nouns can join
function executeSplit():
    // Alternative is to use a split state contract that can't be updated once split is executed
    claimMerkleRoot = calculateClaimMerkle()
    setEscrowsSplitExecuted()

    (newDAOTreasury, newToken) = deploySplitDAO(claimMerkleRoot)
    sendProRataTreasury(newDAOTreasury, escrowCount, adjustedTotalSupply())
    splitEndTimestamp[splitProposalId] = block.timestamp + splitPeriodDuration

    splitProposalId += 1

// joins New DAO with tokenIds directly, without parking in escrow; only works during the split period
function joinSplit(tokenIds):
    require splitPeriodActive()

    nouns.transferFrom(msg.sender, timelock, tokenIds)
    sendProRataTreasury(newDAOTreasury, tokenIds.length, adjustedTotalSupply())
    newToken.claimTokensImmediately(msg.sender, tokenIds)

function deploySplitDAO(claimMerkleRoot):
    desc = new Descriptor(art: originalNounsArt)
    token = new NounsToken(descriptor: desc, originalNounsDAO: dao, originalNounsToken: nouns, splitEndTimestamp[splitProposalId], claimMerkleRoot, escrowClaimCount: escrowCount[splitProposalId])
    auction = new AuctionHouse(paused: true)
    dao = new DAO(vetoer: address(0))
    treasury = new Executor()
    return (treasury, token)

function sendProRataTreasury(newDAOTreasury, tokenCount, totalSupply):
    ethToSend = timelock.balance * tokenCount / totalSupply
    timelock.sendETHToNewDAO(newDAO, ethToSend)

    for erc20 in whitelistedERC20s:
        tokensToSend = erc20.balanceOf(timelock) * tokenCount / totalSupply
        timelock.sendERC20ToNewDAO(newDAO, erc20, tokensToSend)

function splitPeriodActive():
    if splitProposalId == 0:
        return false

    return block.timestamp <= splitEndTimestamp[splitProposalId - 1]

function adjustedTotalSupply():
    return nouns.totalSupply() - nouns.balanceOf(timelock)

function calculateClaimMerkle():
    leaves = []
    for escrow in escrowArrays[splitProposalId]:
        tokenIds = getEscrowTokenIds(escrow)
        for tokenId in tokenIds:
            leaves.push(calculateLeaf(escrow.owner, tokenId))

    return calculateRoot(leaves)

function setEscrowsSplitExecuted():
    for escrow in escrowArrays[splitProposalId]:
        escrow.setSplitExecuted()
```

## Escrow

```jsx
constructor(owner_):
    this.owner = owner_

function delegate(to):
    nouns.delegate(to)

function castRefundableVote(proposalId, support):
    dao.castRefundableVote(proposalId, support)

function setSplitExecuted():
    require msg.sender == dao
    splitExecuted = true

// lets owners pull out prior to split execution; once split is executed lets the DAO take the nouns
function returnToOwner(tokenIds):
    require msg.sender == dao
    require !splitExecuted

    nouns.transferFrom(this, owner, tokenIds)

// anyone can send tokens to the DAO once the split has been executed
function sendToDAO(tokenIds):
    require splitExecuted
    nouns.tranferFrom(this, dao, tokenIds)
```

## New Token

```jsx
constructor(originalNounsDAO_, originalNounsToken_, splitEndTimestamp_, merkleRoot_, escrowClaimCount_):
    this.originalNounsDAO = originalNounsDAO_
    this.originalNounsToken = originalNounsToken_
    this.splitEndTimestamp = splitEndTimestamp_
    this.merkleRoot = merkleRoot_
    this.remainingTokensToClaim = escrowClaimCount_

// owners of Nouns that were put in escrow use this function to claim their New DAO tokens
function claimTokensFromEscrow(tokenIds, merkleProofs):
    for tokenId in tokenIds:
        proof = merkleProofs[tokenId]
        validateProof(msg.sender, tokenId, proof)
        seeds[tokenId] = originalNounsToken.seeds[tokenId]
        _mint(msg.sender, tokenId)

    remainingTokensToClaim -= tokenIds.length

// owners that join a split during the split period can claim immediately
function claimTokensImmediately(owner, tokenIds):
    require msg.sender == originalNounsDAO
    require block.timestamp <= this.splitEndTimestamp

    for tokenId in tokenIds:
        seeds[tokenId] = originalNounsToken.seeds[tokenId]
        _mint(owner, tokenId)
```

## New DAO

```jsx
// vanilla ragequit, protects splitters from majority bullying in New DAO where they might not be able to
// reach split threshold to escape the bullying
function ragequit(tokenIds):
    // reverts if owner didn't approve DAO to transfer their tokens
    token.transferFrom(msg.sender, timelock, tokenIds)
    // sends fair share of ETH and ERC20s, using the number of quitting tokens out of total supply
    sendFairShare(msg.sender, tokenIds.length)

// giving New DAO members time to claim their tokens and get organized before their governance begins
function propose(...):
    require(tokenContract.remainingTokensToClaim() == 0 || block.timestamp > daoCreated + 1 month)
```

## Timelock V2

```jsx
constructor(daoProxy):
    admin = daoProxy

function upgrade(newImplementation):
    require msg.sender == address(this)
    logic = newImplementation

function sendETHToNewDAO(newDAO, ethToSend):
    require msg.sender == admin
    sent = newDAO.send(ethToSend)
    require sent

function sendERC20ToNewDAO(newDAO, erc20, tokensToSend):
    require msg.sender == admin
    erc20.safeTransfer(newDAO, tokensToSend) // reverts if send failed
```
