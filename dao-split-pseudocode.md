## Merged version: successful and failed proposals

### DAO

```jsx
~~~ SIGNALING ~~~

// works from when voting starts, and until the proposal is executed
function signalSplitIfSuccessful(proposalId, tokenIds):
    require state(proposalId) in [Active, ObjectionPeriod, Successful, Queued]
    require !proposal.splitExecuted

    escrow = new SuccessfulProposalsEscrow(owner: msg.sender, proposalId)
    nouns.transferFrom(msg.sender, escrow, tokenIds)
    splitSignalCount[Successful][proposalId] += tokenIds.lengh
    splitTokenIds[Successful][proposalId][msg.sender].pushMany(tokenIds)

    if backingVotesCount(proposalId) <= proposal.threshold:
        cancel(proposalId)

// only works during the Pending period
// failed means Defeated or Vetoed
function signalSplitIfFailed(proposalId, tokenIds):
    require state(proposalId) == Pending

    escrow = new FailedProposalsEscrow(owner: msg.sender, proposalId)
    nouns.transferFrom(msg.sender, escrow, tokenIds)
    splitSignalCount[Failed][proposalId] += tokenIds.lengh
    splitTokenIds[Failed][proposalId][msg.sender] = tokenIds

~~~ EXECUTE SPLIT ~~~

function executeSplitIfSuccessful(proposalId):
    require isProposalExecutable(proposalId)
        && isSuccessfulProposalThresholdMet(proposalId)
    require !proposal.splitExecuted

    proposal.splitExecuted = true

    splitCount = splitSignalCount[Successful][proposalId]
    newDAOTreasury = deploySplitDAO(proposalId, splitCount)
    sendProRataTreasury(newDAOTreasury, splitCount, nouns.totalSupply)

function executeSplitIfFailed(proposalId):
    require state(proposalId) in [Defeated, Vetoed] && !isSplitExpired(proposalId)
    require isFailedProposalThresholdMet(proposalId)
    require !proposal.splitExecuted

    proposal.splitExecuted = true

    splitCount = splitSignalCount[Failed][proposalId]
    newDAOTreasury = deploySplitDAO(proposalId, splitCount)
    sendProRataTreasury(newDAOTreasury, splitCount, nouns.totalSupply)

~~~ SPLIT HELPERS ~~~

function isSuccessfulProposalThresholdMet(proposalId):
    signalCount = splitSignalCount[Successful][proposalId]
    threshold = proposal.totalSupply * splitThresholdBPs / 10_000
    return signalCount >= threshold

function isFailedProposalThresholdMet(proposalId):
    signalCount = splitSignalCount[Failed][proposalId]
    threshold = proposal.totalSupply * splitThresholdBPs / 10_000
    return signalCount >= threshold

// TODO: this can be used for successful prop as well
// if we agree to transition to Queued state without a tx
function isFailedProposalSplitExpired(proposalId):
    return block.number > proposal.endBlock + splitGracePeriod

function isProposalExecutable(proposalId):
    return state(proposalId) == Queued && block.timestamp >= eta && block.timestamp < proposal.expirationTime

~~~ CHANGES TO EXISTING FUNCTIONG ~~~

function execute(proposalId):
    // require: there is not pending split to execute
    if isSuccessfulProposalThresholdMet(proposalId) && !proposal.splitExecuted && !state(proposalId) != Expired:
        revert

    // continue same logic as today

function veto(proposalId):
    require !(state(proposal) in [Pending, Executed])

    // continue same logic as today

```

### SuccessfulProposalsEscrow

```jsx
// TODO add protections against DAO version change by attacker

constructor(owner, proposalId)

function changeDelegate(newDelegate):
    require msg.sender == owner
    require !(newDelegate in dao.proposal(proposalId).proposersOrSigners)

    nouns.delegate(newDelegate)

function withdrawNouns(tokenIds, to):
    // split can't be active
    if (state(proposalId) in [Active, ObjectionPeriod]) ||
        (state(proposalId) == Queued && !dao.proposal(proposalId).splitExecuted):
        revert

    if msg.sender == dao:
        require state(proposalId) == Executed
            && dao.proposal(proposalId).splitExecuted
        nouns.transferFrom(this, to, tokenIds)

    else if msg.sender == owner:
        require !dao.proposal(proposalId).splitExecuted
            && dao.state(proposalId) in [Canceled, Executed, Expired]
        nouns.transferFrom(this, to, tokenIds)

```

### FailedProposalEscrow

```jsx
constructor(owner, proposalId)

function changeDelegate(newDelegate):
    require msg.sender == owner
    require block.number > dao.proposal(proposalId).voteSnapshotBlock

    nouns.delegate(newDelegate)

function castRefundableVote(voteProposalId, support):
    if voteProposalId == proposalId:
        require support == 1 // must vote For
    else:
        require msg.sender == owner

    dao.castRefundableVote(voteProposalId, support)

function withdrawNouns(tokenIds, to):
    state = state(proposalId)
    if state == Canceled:
        require msg.sender == owner
        nouns.transferFrom(this, to, tokenIds)
        return

    require state in [Defeated, Vetoed]

    if msg.sender == dao:
        require dao.proposal(proposalId).splitExecuted
        nouns.transferFrom(this, to, tokenIds)
        return

    require msg.sender == owner

    require dao.isFailedProposalSplitExpired(proposalId)
            || !dao.isFailedProposalThresholdMet(proposalID)

    nouns.transferFrom(this, to, tokenIds)

```

### New DAO

```jsx
function constructor():
	daoCreated = block.timestamp

function propose(...):
	require(tokenContract.remainingTokensToClaim() == 0 || block.timestamp > daoCreated + 1 month)

```

### New Token

```jsx
function constructor(descriptor, originalNounsDAO, originalNounsToken, proposalId, tokensToClaim, successfulOrFailedProposal_):
	// save to immutables
    remainingTokensToClaim = tokensToClaim
    successfulOrFailedProposal = successfulOrFailedProposal_

function claimTokens(tokenIds[]):
	require dao.proposals(proposalId).splitExecuted
	require tokenIds in dao.splitTokenIds[successfulOrFailedProposal][proposalId][msg.sender]

	for tokenId in tokenIds:
		seeds[tokenId] = originalNounsToken.seeds[tokenId]
		_mint(msg.sender, tokenId)

	remainingTokensToClaim -= tokenIds.length

```

### TimelockV2

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

### SplitState

We can add this contract as a 'state escrow' that mitigates the risk of the DAO being upgraded to provide false information.