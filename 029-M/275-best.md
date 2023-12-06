Bright Fossilized Seagull

medium

# Vesting schedule not work for founders with over 50% ownership

## Summary

A vesting schedule calculation flaw allows a founder with more than 50% ownership to bypass the intended vesting period, receiving all NFTs immediately.

## Vulnerability Detail

When the DAO is created, depends on each founder ownership percentage, he will receive a NFT for each percentage that he owns. There is a vesting schedule for the founders to receive the NFTs. 

For example, assume that reservedUntilTokenId = 0, if the founder percentage is 25, then the schedule is 100 / 25 = 4. The distribution occurs every fourth NFT: 0, 4, 8, 12, 16, ... 96,  meaning the founder waits for three NFTs to be minted before receiving another.
        
        // Compute the vesting schedule
        uint256 schedule = 100 / founderPct;

        // Used to store the base token id the founder will recieve
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
            
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L158-L176

The issue arises when a founder's ownership exceeds 50%. In such cases, the schedule calculation (100 / founderPct) rounds down to 1. This results in the founder receiving every NFT sequentially (0, 1, 2, 3, ...), effectively nullifying the vesting schedule.

        // Compute the vesting schedule
        uint256 schedule = 100 / founderPct;

In this scenario, the vesting schedule ceases to function as intended.

## Impact

This flaw allows a founder with more than 50% ownership to acquire all NFTs immediately upon launch, undermining the vesting schedule's purpose.

## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L158-L176

## Tool used

Manual Review

## Recommendation

Consider adjusting the schedule calculation to account for rounding down scenarios:. Can consider add 1 to the schedule:

```diff
- uint256 schedule = 100 / founderPct;
+ uint256 schedule = (100 / founderPct) + 1;
```