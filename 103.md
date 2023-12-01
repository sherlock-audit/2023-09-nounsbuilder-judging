Skinny Frost Puppy

high

# attacker can win all auctions with minimum bid by performing returnbomb attack

## Summary
function `createBid()` creates a higher bid for the current token and in doing so, it would return the last highest bidder ETH by calling `_handleOutgoingTransfer()`. the issue is that code use `call()` to send ETH to previous bidder and a malicious bidder can return a gas bomb to Auction contract and cause revert for the whole transaction. so attacker can do a minimum bidding by his malicious contract and prevent others from bidding higher and win all the auctions.

## Vulnerability Detail
This is part of the `_createBid()` code:
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
as you see in last line code calls `_handleOutgoingTransfer()` to send back previous bidder funds. This is `_handleOutgoingTransfer()` code:
```javascript
    function _handleOutgoingTransfer(address _to, uint256 _amount) private {
        // Ensure the contract has enough ETH to transfer
        if (address(this).balance < _amount) revert INSOLVENT();

        // Used to store if the transfer succeeded
        bool success;

        assembly {
            // Transfer ETH to the recipient
            // Limit the call to 50,000 gas
            success := call(50000, _to, _amount, 0, 0, 0, 0)
        }

        // If the transfer failed:
        if (!success) {
            // Wrap as WETH
            IWETH(WETH).deposit{ value: _amount }();

            // Transfer WETH instead
            bool wethSuccess = IWETH(WETH).transfer(_to, _amount);

            // Ensure successful transfer
            if (!wethSuccess) {
                revert FAILING_WETH_TRANSFER();
            }
        }
    }
```
as you can see code use `call(50000, _to, _amount, 0, 0, 0, 0)` to send ETH to the last bidder, even so code hardcoded 50K gas to prevent callee to use gas and cause OOG, but the callee can perform returnbomb attack and cause revert for whole transaction. in return bomb attack, callee returns very large return data and when caller tries to copy that data to memory, because of memory expansions gas, the callee will get OOG error and revert. it's important to note that callee send back arbitrary length data with 50K gas, and limiting the callee gas to 50K gas doesn't prevent this attack. you can read more about this in: https://github.com/nomad-xyz/ExcessivelySafeCall

also a sample returnbomb code: https://gist.github.com/pcaversaccio/3b487a24922c839df22f925babd3c809

## Impact
attacker can win all auctions with minimum bid. (attacker can deny others from bidding higher than him)

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L517-L543
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L226-L227

## Tool used
Manual Review

## Recommendation
use EcessivelySafeCall library to send back the ETH. or allow last bidder to pull his funds later.