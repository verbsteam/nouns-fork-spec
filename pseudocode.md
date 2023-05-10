# Nouns Fork Pseudocode

## This files is outdated; please refer to the spec only for now. An update is coming soon!

---

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
    require escrowCount[splitProposalId] > splitThreshold()

    splitProposalExecuted[splitProposalId] = true

    // Alternative is to use a split state contract that can't be updated once split is executed
    claimMerkleRoot = calculateClaimMerkle()

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

function splitThreshold():
    return adjustedTotalSupply() * splitThresholdBPs / 10_000
```

## Escrow

```jsx
constructor(owner_, splitProposalId_):
    this.owner = owner_
    this.splitProposalId = splitProposalId_
    nouns.delegate(owner_)

function delegate(to):
    nouns.delegate(to)

function castRefundableVote(proposalId, support):
    dao.castRefundableVote(proposalId, support)

// lets owners pull out prior to split execution; once split is executed lets the DAO take the nouns
function returnToOwner(tokenIds):
    require msg.sender == dao
    require !dao.splitProposalExecuted(splitProposalId)

    nouns.transferFrom(this, owner, tokenIds)

// anyone can send tokens to the DAO once the split has been executed
// maybe set the DAO as approved address to transfer instead
function sendToDAO(tokenIds):
    require dao.splitProposalExecuted(splitProposalId)
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
    require block.timestamp <= this.splitEndTimestamp // this is to prevent OG DAO upgrading contracts and trying to mint new tokens

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
    token.transferFrom(msg.sender, timelock, tokenIds) // maybe burn instead?
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
