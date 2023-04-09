# Split Proposal

## High level spec

- any Nouner can create a new split proposal, as long as there isn't already an active one.
- the proposal to split is in 'escrow period' from creation until threshold is reached, or until all Nouns pull out of escrow.
- once split threshold is reached anyone can execute the split, moving the DAO into a 'split period', during which Nouners can split with more nouns immediately into the New DAO, without parking in escrow first; the split period duration is a DAO-configured variable, e.g. set to 7 days.
- during the 'split period' OG DAO proposals cannot be executed.
- once a split proposal enters the split period, the split cannot be cancelled, and escrowed Nouns are committed to the split and cannot be pulled out.
- New DAO support 'vanilla ragequit', i.e. any Nouner can immediately quit any time with their fair share; this design protects Nouners against a malcious actor or any majority bullying in New DAO.
