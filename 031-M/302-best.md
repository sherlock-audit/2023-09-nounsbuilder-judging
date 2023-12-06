Ripe Pecan Wolf

medium

# WETH Transfers Will Not Work For Migrated DAOs

## Summary

The `Auction::_handleOutGoingTransfer` function transfers funds to a user when Auctions are settled or the user is outbid. It first tries to transfer ETH if the ETH transfer fails, as a backup, the Auction contract will transfer WETH. When a DAO is migrated to a different chain the WETH transfers will fail.

## Vulnerability Detail

The problem is that the WETH address is set as a private immutable variable upon contract deployment and WETH has different addresses across different chains. When DAOs migrate to a different chain the WETH address does not change and the transfer functionality will break.

## Impact

WETH transfers will fail and users may lose funds

## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L49

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L531-#L540
## Tool used

Manual Review

## Recommendation

Create an access restricted function that can change the address of WETH

