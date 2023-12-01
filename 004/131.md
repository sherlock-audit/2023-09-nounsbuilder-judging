Skinny Frost Puppy

medium

# changing reservePrice or auction duration doesn't effect the current auction

## Summary
DAO owner can set new value for `reservePrice` or auction's  `duration` when the Auction contract is paused. the issue is that those changes doesn't effect the current auction and the current auction can settle with lower price than new `reservePrice` because only the first bid price is checked with `reservePrice` while code should check the final auction highest bid with `reservePrice`.

## Vulnerability Detail
These functions that are in charge of changing `reservePrice` and auction's `duration`, as you can see the  DAO owner can change those value when the Auction is paused:
```javascript
    function setDuration(uint256 _duration) external onlyOwner whenPaused {
        settings.duration = SafeCast.toUint40(_duration);

        emit DurationUpdated(_duration);
    }

    function setReservePrice(uint256 _reservePrice) external onlyOwner whenPaused {
        settings.reservePrice = _reservePrice;

        emit ReservePriceUpdated(_reservePrice);
    }
```
This is `createBid()` and functions, as you can see it can be called even if Auction is paused.
```javascript
function createBid(uint256 _tokenId) external payable nonReentrant {
        currentBidReferral = address(0);
        _createBid(_tokenId);
    }
```

This is part of `creatBid()` that checks bid prices, as you can see the bid price only checked with `reservePrice` only if its the first bid.
```javascript
 // If this is the first bid:
        if (lastHighestBidder == address(0)) {
            // Ensure the bid meets the reserve price
            if (msgValue < settings.reservePrice) {
                revert RESERVE_PRICE_NOT_MET();
            }

            // Else this is a subsequent bid:
        } else {
            // Used to store the minimum bid required
            uint256 minBid;

            // Cannot realistically overflow
            unchecked {
                // Compute the minimum bid
                minBid = lastHighestBid + ((lastHighestBid * settings.minBidIncrement) / 100);
            }

            // Ensure the incoming bid meets the minimum
            if (msgValue < minBid) {
                revert MINIMUM_BID_NOT_MET();
            }
            // Ensure that the second bid is not also zero
            if (minBid == 0 && msgValue == 0 && lastHighestBidder != address(0)) {
                revert MINIMUM_BID_NOT_MET();
            }

            // Refund the previous bidder
            _handleOutgoingTransfer(lastHighestBidder, lastHighestBid);
        }
```

so if DAO owners changes the auction's `reservePrice` or `duration` , those  changes won't effect the current auction and current auction can be bought with lower price than `reservePrice`. 

## Impact
changes for auction properties doesn't effect the current auction even when the Auction contract is paused.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L199-L228
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L408-L421

## Tool used
Manual Review

## Recommendation
check the auction bid price with `reservePrice` in each bid and in settelment. 
