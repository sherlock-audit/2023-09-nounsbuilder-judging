Orbiting Emerald Yeti

medium

# Token#updateFounders fails to properly clear tokenRecipient mapping causing improper token distribution

## Summary

Token#updateFounders is meant to clear the token recipient mapping then add the newly requested founders. When adding founders, the token number begins with reservedUntilTokenId. The issue is that when clearing the mapping, it starts at zero. This disparity leads to improper token distribution since a majority of time, the original distribution won't be cleared and a new distribution will be added.

## Vulnerability Detail

[Token.sol#L161-L175](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L161-L175)

    uint256 baseTokenId = reservedUntilTokenId;

    // For each token to vest:
    for (uint256 j; j < founderPct; ++j) {
        // Get the available token id
        baseTokenId = _getNextTokenId(baseTokenId);

        // Store the founder as the recipient
        tokenRecipient[baseTokenId] = newFounder;

        emit MintScheduled(baseTokenId, founderId, newFounder);

        // Update the base token id
        baseTokenId = (baseTokenId + schedule) % 100;
    }

When adding founders to a token, `baseTokenId` begins with `reservedUntilTokenId`. It then iterates through the founder percentage, adding each token spaced out according to the schedule.

[Token.sol#L412-L427](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L412-L427)

    uint256 baseTokenId; @audit starts at 0

    for (uint256 j; j < cachedFounder.ownershipPct; ++j) {
        // Get the next index that hasn't already been cleared
        while (clearedTokenIds[baseTokenId] != false) {
            baseTokenId = (++baseTokenId) % 100;
        }

        delete tokenRecipient[baseTokenId];
        clearedTokenIds[baseTokenId] = true;

        emit MintUnscheduled(baseTokenId, i, cachedFounder);

        // Update the base token id
        baseTokenId = (baseTokenId + schedule) % 100;
    }

When removing the founders it uses a similar methodology with one major difference. It starts at 0 rather than `reservedUntilTokenId`. This leads to a mismatch of tokens being removed and added and the mapping isn't cleared correctly. The following example shows this:

Assume `reservedUntilTokenId` is 50 and founder ownership is 5%. When adding the founder it would result in the following tokens: 51 (50 + 1), 71 (51 + 20), 91 (71 + 20), 11 (91 + 20 % 100), 31 (11 + 20).

When removing however it will clear: 1 (0 + 1), 21 (1 + 20), 41 (21 + 20), 61 (41 + 20), 81 (61 + 20)

In the scenario shown above, updating the founder rewards would cause a massive over allocation to the founder (+100%).

## Impact

Founder receives too many tokens, harming token holders by over-dilution and loss of treasury funding

## Code Snippet

[Token.sol#L375-L437](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L375-L437)

## Tool used

Manual Review

## Recommendation

Start with the proper index when removing rather than 0.