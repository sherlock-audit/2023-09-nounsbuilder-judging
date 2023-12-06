Cheerful Boysenberry Shell

medium

# `Auction.upgradeTo` and `Auction.upgradeToAndCall` might mess up the logic implementation of Auction

## Summary
While calling `Auction.upgradeTo` and `Auction.upgradeToAndCall`, the only requirement is that Auction is paused, but when Auction.sol is paused, there might be a ongoing auction, in such case, replacing the implementation of Auction might mess up the logic of Auction

## Vulnerability Detail
When calling [Auction.upgradeTo](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/lib/proxy/UUPS.sol#L47-L50) and [Auction.upgradeToAndCall](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/lib/proxy/UUPS.sol#L55-L57), the two functions will call [Auction._authorizeUpgrade](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L552-L555) to ensures the caller is authorized to upgrade the contract and the new implementation is valid.

```solidity
    function _authorizeUpgrade(address _newImpl) internal view override onlyOwner whenPaused {
        // Ensure the new implementation is registered by the Builder DAO
        if (!manager.isRegisteredUpgrade(_getImplementation(), _newImpl)) revert INVALID_UPGRADE(_newImpl);
    }
```
As show above, the function only checks if the Auction contract has been paused, but it doesn't check if there is an ongoing auction.

## Impact
If there is an ongoing auction in the contract, the new implementation of `Auction.createBid` and `Auction.createBidWithReferral` might be different from the old one. In such case, it might be wrong.

## Code Snippet

## Tool used

Manual Review

## Recommendation
 ```diff
 diff --git a/nouns-protocol/src/auction/Auction.sol b/nouns-protocol/src/auction/Auction.sol
index f3b1d63..6798f24 100644
--- a/nouns-protocol/src/auction/Auction.sol
+++ b/nouns-protocol/src/auction/Auction.sol
@@ -149,7 +149,7 @@ contract Auction is IAuction, VersionedContract, UUPS, Ownable, ReentrancyGuard,
     /// @notice Creates a bid for the current token
     /// @param _tokenId The ERC-721 token id
     function createBid(uint256 _tokenId) external payable nonReentrant {
-        currentBidReferral = address(0);
+        currentBidReferral = msg.sender;
         _createBid(_tokenId);
     }

@@ -431,7 +431,11 @@ contract Auction is IAuction, VersionedContract, UUPS, Ownable, ReentrancyGuard,
     /// @param _percentage The new percentage
     function setMinimumBidIncrement(uint256 _percentage) external onlyOwner whenPaused {
         if (_percentage == 0) {
-            revert MIN_BID_INCREMENT_1_PERCENT();
+            revert IN_AUCTION();
+        }
+
+        if(auction.settled == false) {
+            revert
         }

         settings.minBidIncrement = SafeCast.toUint8(_percentage);
diff --git a/nouns-protocol/src/auction/IAuction.sol b/nouns-protocol/src/auction/IAuction.sol
index 5dcf4c8..30fa607 100644
--- a/nouns-protocol/src/auction/IAuction.sol
+++ b/nouns-protocol/src/auction/IAuction.sol
@@ -103,6 +103,7 @@ interface IAuction is IUUPS, IOwnable, IPausable {
     /// @dev Thrown if the rewards total is greater than 100%
     error INVALID_REWARD_TOTAL();

+    error IN_AUCTION();
     ///                                                          ///
     ///                          STRUCTS                         ///
     ///                                                          ///
```
