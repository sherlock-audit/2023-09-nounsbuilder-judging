Fit Hazelnut Donkey

medium

# ` auction.highestBid` may not really be highest bid

## Summary
msgValue of current bid isn't compared with `lastHighestBid` before `auction.highestBid`  is updated in `_createBid()` function.

## Vulnerability Detail
The _createBid() function fails to compare the `lastHighestBid` with msgValue of current bid, to ensure msgValue of current bid is indeed higher than `lastHighestBid` before updating the  `auction.highestBid` with msg.value of current bid.

The problem lies [here](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L179), since there's no check like i said above, the latest bid will be the  `auction.highestBid` instead of the true highest bid.



Same applies to `auction.highestBidder` in this [line](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L182)
## Impact
The bid system is flawed. 
` auction.highestBid` might not really be highest bid.
latest bid instead of highest bid will win the auction.


## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L179

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L182
## Tool used

Manual Review

## Recommendation
 compare the `lastHighestBid` with msgValue of current bid, to ensure msgValue of current bid is indeed higher than `lastHighestBid` before updating the  `auction.highestBid` with msg.value of current bid.