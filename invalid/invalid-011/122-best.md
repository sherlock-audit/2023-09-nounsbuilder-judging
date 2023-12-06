Skinny Frost Puppy

high

# there is no minimum limit for auction duration and it can cause unlimited token burn and mint

## Summary
function `_createAuction()` in Auction contract creates an auction for the next token and the end time of the auction is set to `block.timestamp + duration` which is based on `duration` that founders set for the Auction. funders allowed to set values that fit into `uint40`. if the `endTime <= block.timstamp` then the auction is finished and can be settled. the issue is that if founders set a zero value for auction's duration, then the end time would set to `block.timestamp` and auction would be finished before even started. attacker can use this and call `_createAuction()` multiple times and cause Auction contract to burn previous auction token and mint new token which will mint tokens for founders from vesting allocation. 

I believe this has High severity because founders with vesting allocation have incentive to perform the attack, they can mint unlimited tokens for themself in one short time and sell them in the market and perform rug pull. 

the issue will happen to some extend if the auction time was low like some seconds. it's not logical to create auctions that last some seconds. so overall Nounce builder should set some minimum limit for auction duration like 30 minute so users who interact with the DAO can't be rugged easily by founders or leaked private keys.

## Vulnerability Detail
This is part of `_createAuction()` code that calculates the end time, as you can see `endTime` is set to `block.timestamp + duration` if duration was zero then auction would be in finished state right from auction creation.
```javascript
            // Cache the current timestamp
            uint256 startTime = block.timestamp;

            // Used to store the auction end time
            uint256 endTime;

            // Cannot realistically overflow
            unchecked {
                // Compute the auction end time
                endTime = startTime + settings.duration;
            }

            // Store the auction start and end time
            auction.startTime = uint40(startTime);
            auction.endTime = uint40(endTime);
```

This is the `setDuration()` function which founders can used to set auction durations, as you can see founders allowed to set duration as low as zero:
```javascript
    function setDuration(uint256 _duration) external onlyOwner whenPaused {
        settings.duration = SafeCast.toUint40(_duration);

        emit DurationUpdated(_duration);
    }
```

This is part of `settleAuction()` code, as you can see if `block.timestamp >= endTime` then auction is finished:
```javascript
    function _settleAuction() private {
        Auction memory _auction = auction;

        if (auction.settled) revert AUCTION_SETTLED();

        if (_auction.startTime == 0) revert AUCTION_NOT_STARTED();

        // Ensure the auction is over
        if (block.timestamp < _auction.endTime) revert AUCTION_ACTIVE();
```

so overall code allows for 0 duration auction and low duration auctions. This will cause security risk for DAO users as they need to trust the founders to not do any malicious action by manipulating auction duration also users need to be worry about private key leakage or changing duration by mistake. DAO builder should prevent this risks by setting a minimum limit for auction duration.


This is the POC for this issue:
1. DAO founders for any reason want to decrease auction duration so they set auction duration to `0`, which is allowable by Auction contract.
2. now when the current Auction finished, attacker can call `settleCurrentAndCreateNewAuction()` to settle the finished auction and create a new auction.
3. code would settle the finished auction and would try to create new auction. the auction end time would be `uint40(block.timestamp + 0) = block.timestamp`, so the created auction will be in finished state because when `endTime <= block.timestamp` auction can be settled in `settleAuction()` function.
4. now attacker can call `settleCurrentAndCreateNewAuction()` again create a new auction and the same thing will happen. the auction contract would burn the previous auction token and create a new auction that is expired.
5. attacker can create a malicious contract that calls `settleCurrentAndCreateNewAuction()` in a loop and create hundreds of expired auction and by doing so two thing would happen: a) tokens that are supposed to be auctioned will be burned. b) Founders would receive a lot of NFT tokens for their vesting allocations.
6. in the end in short amount of time attacker can burn arbitrary amount of tokens and  would mint arbitrary amount of tokens for founders.

This attack would be possible to some extend if auction duration was low. in that case the price that came from auction won't show the real value of the NFT and users would buy NFTs with different prices and the NFT distribution and auction won't be fair.

## Impact
attacker or malicious founder can cause fund loss for users by minting a lot of tokens for founder and burning auction tokens. the token price will go to 0 because of unlimited token mints.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L297-L312

## Tool used
Manual Review

## Recommendation
set a minimum limit for auction duration.