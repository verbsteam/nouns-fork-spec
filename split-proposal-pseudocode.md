# Split Proposal

## High level spec

- any Nouner can create a new split proposal, as long as there isn't already an active one.
- the proposal to split is in 'escrow period' from creation until threshold is reached, or until all Nouns pull out of escrow.
- once split threshold is reached anyone can execute the split, moving the DAO into a 'split period', during which Nouners can split with more nouns immediately into the New DAO, without parking in escrow first; the split period duration is a DAO-configured variable, e.g. set to 7 days.
- during the 'split period' OG DAO proposals cannot be executed.
- once a split proposal enters the split period, the split cannot be cancelled, and escrowed Nouns are committed to the split and cannot be pulled out.
- New DAO support 'vanilla ragequit', i.e. any Nouner can immediately quit any time with their fair share; this design protects Nouners against a malcious actor or any majority bullying in New DAO.

## DAO

```jsx
// puts tokens in escrow
function signalSplit(tokenIds):
    // once split is active we don't allow escrows; instead use joinSplit
    require !splitPeriodActive()

    escrow = getOrCreateEscrow(owner: msg.sender)
    nouns.transferFrom(msg.sender, escrow, tokenIds)
    escrowCount += tokenIds.length

// takes tokens out of escrow
function unsignalSplit(tokenIds, to):
    require !splitPeriodActive()

    escrow = getEscrow(owner: msg.sender)
    escrow.returnToOwner(tokenIds)
    escrowCount -= tokenIds.length


// creates New DAO, sends it money, and kicks off 'split period' when more Nouns can join
function executeSplit():
    // TODO how do I get all token Ids? does it make sense to build the tree as escrows come in and out?
    calculateMerkleTree()
    (newDAOTreasury, newToken) = deploySplitDAO()
    sendProRataTreasury(newDAOTreasury, escrowCount, adjustedTotalSupply())
    splitEndTimestamp = block.timestamp + splitPeriodDuration


// joins New DAO with tokenIds directly, without parking in escrow; only works during the split period
function joinSplit(tokenIds):
    require splitPeriodActive()

    nouns.transferFrom(msg.sender, timelock, tokenIds)
    sendProRataTreasury(newDAOTreasury, tokenIds.length, adjustedTotalSupply())
    newToken.claimTokensImmediately(msg.sender, tokenIds)

function deploySplitDAO():
    desc = new Descriptor(art: originalNounsArt)
    token = new NounsToken(descriptor: desc, originalNounsDAO: dao, originalNounsToken: nouns, splitEndTimestamp)
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
    return block.timestamp <= splitEndTimestamp

function adjustedTotalSupply():
    return nouns.totalSupply() - nouns.balanceOf(timelock)
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
constructor(originalNounsDAO_, originalNounsToken_, splitEndTimestamp_):
    this.originalNounsDAO = originalNounsDAO_
    this.originalNounsToken = originalNounsToken_
    this.splitEndTimestamp = splitEndTimestamp_

// owners of Nouns that were put in escrow use this function to claim their New DAO tokens
function claimTokensFromEscrow(tokenIds, merkleProofs):
    for tokenId in tokenIds:
        proof = merkleProofs[tokenId]
        validateProof(msg.sender, tokenId, proof)
        seeds[tokenId] = originalNounsToken.seeds[tokenId]
        _mint(msg.sender, tokenId)

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
    // TODO something like this:
    // require(tokenContract.remainingTokensToClaim() == 0 || block.timestamp > daoCreated + 1 month)
```
