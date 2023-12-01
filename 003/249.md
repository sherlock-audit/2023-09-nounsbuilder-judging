Orbiting Emerald Yeti

medium

# MerkleReserveMinter minting methodology is incompatible with current governance structure and can lead to migrated DAOs being hijacked immediately

## Summary

MerkleReserveMinter allows large number of tokens to be minted instantaneously which is incompatible with the current governance structure which relies on tokens being minted individually and time locked after minting by the auction. By minting and creating a proposal in the same block a user is able to create a proposal with significantly lower quorum than expected. This could easily be used to hijack the migrated DAO.

## Vulnerability Detail

[MerkleReserveMinter.sol#L154-L167](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L154-L167)

    unchecked {
        for (uint256 i = 0; i < claimCount; ++i) {
            // Load claim in memory
            MerkleClaim memory claim = claims[I];

            // Requires one proof per tokenId to handle cases where users want to partially claim
            if (!MerkleProof.verify(claim.merkleProof, settings.merkleRoot, keccak256(abi.encode(claim.mintTo, claim.tokenId)))) {
                revert INVALID_MERKLE_PROOF(claim.mintTo, claim.merkleProof, settings.merkleRoot);
            }

            // Only allowing reserved tokens to be minted for this strategy
            IToken(tokenContract).mintFromReserveTo(claim.mintTo, claim.tokenId);
        }
    }

When minting from the claim merkle tree, a user is able to mint as many tokens as they want in a single transaction. This means in a single transaction, the supply of the token can increase very dramatically. Now we'll take a look at the governor contract as to why this is such an issue.

[Governor.sol#L184-L192](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/governance/governor/Governor.sol#L184-L192)

        // Store the proposal data
        proposal.voteStart = SafeCast.toUint32(snapshot);
        proposal.voteEnd = SafeCast.toUint32(deadline);
        proposal.proposalThreshold = SafeCast.toUint32(currentProposalThreshold);
        proposal.quorumVotes = SafeCast.toUint32(quorum());
        proposal.proposer = msg.sender;
        proposal.timeCreated = SafeCast.toUint32(block.timestamp);

        emit ProposalCreated(proposalId, _targets, _values, _calldatas, _description, descriptionHash, proposal);

[Governor.sol#L495-L499](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/governance/governor/Governor.sol#L495-L499)

    function quorum() public view returns (uint256) {
        unchecked {
            return (settings.token.totalSupply() * settings.quorumThresholdBps) / BPS_PER_100_PERCENT;
        }
    }

When creating a proposal, we see that it uses a snapshot of the CURRENT total supply. This is what leads to the issue. The setup is fairly straightforward and occurs all in a single transaction:

1) Create a malicious proposal (which snapshots current supply)
2) Mint all the tokens
3) Vote on malicious proposal with all minted tokens

The reason this works is because the quorum is based on the supply before the mint while votes are considered after the mint, allowing significant manipulation of the quorum.

## Impact

DOA can be completely hijacked

## Code Snippet

[MerkleReserveMinter.sol#L129-L173](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L129-L173)

## Tool used

Manual Review

## Recommendation

Token should be changed to use a checkpoint based total supply, similar to how balances are handled. Quorum should be based on that instead of the current supply.