Orbiting Emerald Yeti

high

# Adversary can permanently brick auctions due to precision error in Auction#_computeTotalRewards

## Summary

When batch depositing to ProtocolRewards, the msg.value is expected to match the sum of the amounts array EXACTLY. The issue is that due to precision loss in Auction#_computeTotalRewards this call can be engineered to always revert which completely bricks the auction process.

## Vulnerability Detail

[ProtocolRewards.sol#L55-L65](https://github.com/ourzora/zora-protocol/blob/8d1fe9bdd79a552a8f74b4712451185f6aebf9a0/packages/protocol-rewards/src/ProtocolRewards.sol#L55-L65)

        for (uint256 i; i < numRecipients; ) {
            expectedTotalValue += amounts[i];

            unchecked {
                ++i;
            }
        }

        if (msg.value != expectedTotalValue) {
            revert INVALID_DEPOSIT();
        }

When making a batch deposit the above method is called. As seen, the call with revert if the sum of amounts does not EXACTLY equal the msg.value.

[Auction.sol#L474-L507](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L474-L507)

        uint256 totalBPS = _founderRewardBps + referralRewardsBPS + builderRewardsBPS;

        ...

        // Calulate total rewards
        split.totalRewards = (_finalBidAmount * totalBPS) / BPS_PER_100_PERCENT;

        ...

        // Initialize arrays
        split.recipients = new address[](arraySize);
        split.amounts = new uint256[](arraySize);
        split.reasons = new bytes4[](arraySize);

        // Set builder reward
        split.recipients[0] = builderRecipient;
        split.amounts[0] = (_finalBidAmount * builderRewardsBPS) / BPS_PER_100_PERCENT;

        // Set referral reward
        split.recipients[1] = _currentBidRefferal != address(0) ? _currentBidRefferal : builderRecipient;
        split.amounts[1] = (_finalBidAmount * referralRewardsBPS) / BPS_PER_100_PERCENT;

        // Set founder reward if enabled
        if (hasFounderReward) {
            split.recipients[2] = founderReward.recipient;
            split.amounts[2] = (_finalBidAmount * _founderRewardBps) / BPS_PER_100_PERCENT;
        }

The sum of the percentages are used to determine the totalRewards. Meanwhile, the amounts are determined using the broken out percentages of each. This leads to unequal precision loss, which can cause totalRewards to be off by a single wei which cause the batch deposit to revert and the auction to be bricked. Take the following example:

Assume a referral reward of 5% (500) and a builder reward of 5% (500) for a total of 10% (1000). To brick the contract the adversary can engineer their bid with specific final digits. In this example, take a bid ending in 19. 

    split.totalRewards = (19 * 1,000) / 100,000 = 190,000 / 100,000 = 1

    split.amounts[0] = (19 * 500) / 100,000 = 95,000 / 100,000 = 0
    split.amounts[1] = (19 * 500) / 100,000 = 95,000 / 100,000 = 0

Here we can see that the sum of amounts is not equal to totalRewards and the batch deposit will revert. 

[Auction.sol#L270-L273](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L270-L273)

    if (split.totalRewards != 0) {
        // Deposit rewards
        rewardsManager.depositBatch{ value: split.totalRewards }(split.recipients, split.amounts, split.reasons, "");
    }

The depositBatch call is placed in the very important _settleAuction function. This results in auctions that are permanently broken and can never be settled.

## Impact

Auctions are completely bricked

## Code Snippet

[Auction.sol#L244-L289](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L244-L289)

## Tool used

Manual Review

## Recommendation

Instead of setting totalRewards with the sum of the percentages, increment it by each fee calculated. This way they will always match no matter what.