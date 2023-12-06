Cold Blonde Sparrow

medium

# The `Token::setReservedUntilTokenId()` function does not update the assigned founders in the `tokenRecipient[]` array causing founders being unable to claim some assigned tokens

## Summary

The `Token::setReservedUntilTokenId()` function does not update the assigned founders in the `tokenRecipient[]` array causing founders being unable to claim some assigned tokens.

## Vulnerability Detail

Founders are assigned in the `tokenRecipient[]` array once the token is initialized in the [Token:_addFounders()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L120) function. The position assigned in the `tokenRecipient[]` array depends if the [token is initialized using `reservedUntilTokenId` code line 161](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L161C39-L161C59):

```solidity
File: Token.sol
160:                 // Used to store the base token id the founder will recieve
161:                 uint256 baseTokenId = reservedUntilTokenId;
162: 
163:                 // For each token to vest:
164:                 for (uint256 j; j < founderPct; ++j) {
165:                     // Get the available token id
166:                     baseTokenId = _getNextTokenId(baseTokenId);
167: 
168:                     // Store the founder as the recipient
169:                     tokenRecipient[baseTokenId] = newFounder;
170: 
171:                     emit MintScheduled(baseTokenId, founderId, newFounder);
172: 
173:                     // Update the base token id
174:                     baseTokenId = (baseTokenId + schedule) % 100;
175:                 }
```

The issue arises when the `reservedUntilTokenId` can be updated using [Token::setReservedUntilTokenId()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L486) function but the founders `tokenRecipient[]` will not be updated with the new `reservedUntilTokenId` value. As a result, some assigned tokens to the founder will be unclaimable.

## Impact

Founders will not be able to claim some assigned tokens if the `reservedUntilTokenId` is updated because the assigned `tokenRecipient` is not updated. Please see the next scenario:

1. The token is initialized and one founder is assigned [99 ownershipPct](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L130) and [reservedUntilTokenId zero](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L161).
2. The founder is assigned the array [tokenRecipient[] from 0 to 98](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L169C21-L169C35).
3. The owner sets a new `reservedUntilTokenId to 98` using the [setReservedUntilTokenId()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L486) function.
4. After the auction begins, the founder will only `receive one token`, since during the minting process the [tokenId](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L235) will start at 98 ID and no previous token IDs will be minted for the founder.
5. The token ownership is transferred to the `treasury` causing that a proposal needs to be created if owner wants to update the founders list in order to fix the problem.

The next test reveals that the founder is assigned 99 initially, but ends up receiving only 1 token.

```solidity
// File: Token.t.sol
// forge test --match-test "test_UpdateReserveUntilTokenIdWillNotUpdateTheAssignedFounders" -vvv
//
    function test_UpdateReserveUntilTokenIdWillNotUpdateTheAssignedFounders() public {
        // Founder will not able to get some tokens once the reservedUntilTokenId is updated
        //
        // 1. The token is initialized, one founder and 99 ownershipPct and reserverUntilTokenId zero
        createUsers(1, 1 ether);
        address[] memory wallets = new address[](1);
        uint256[] memory percents = new uint256[](1);
        uint256[] memory vestExpirys = new uint256[](1);
        address founder = otherUsers[0];
        wallets[0] = founder;
        percents[0] = 99;
        vestExpirys[0] = 4 weeks;
        deployWithCustomFounders(wallets, percents, vestExpirys);
        //
        // 2. The founder is assigned the array tokenRecipient[] from 0 to 98
        assertEq(token.totalFounders(), 1);
        assertEq(token.totalFounderOwnership(), 99);
        unchecked {
            for (uint256 i; i < 99; ++i) {
                assertEq(token.getScheduledRecipient(i).wallet, founder);
            }
        }
        //
        // 3. Set a new reservedUntulTokenId = 98 and the founder still will have the same tokenRecipient[0]
        // causing the founder being unable to claim his tokens
        vm.prank(address(founder));
        token.setReservedUntilTokenId(98);
        unchecked {
            for (uint256 i; i < 99; ++i) {
                assertEq(token.getScheduledRecipient(i).wallet, founder);
            }
        }
        //
        // 4. The Auction starts and founder obtain only one token
        vm.prank(address(auction));
        uint256 tokenId = token.mint();
        assertEq(token.balanceOf(founder), 1);
    }
```

## Code Snippet

- [Token::setReservedUntilTokenId()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L486)

## Tool used

Manual review

## Recommendation

The founder [tokenRecipient[]](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L169) array should be updated using the new `reservedUntilTokenId` in the [setReservedUntilTokenId()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L486) function:


```diff
    function setReservedUntilTokenId(uint256 newReservedUntilTokenId) external onlyOwner {
        // Cannot change the reserve after any non reserved tokens have been minted
        // Added to prevent making any tokens inaccessible
        if (settings.mintCount > 0) {
            revert CANNOT_CHANGE_RESERVE();
        }

        // Cannot decrease the reserve if any tokens have been minted
        // Added to prevent collisions with tokens being auctioned / vested
        if (settings.totalSupply > 0 && reservedUntilTokenId > newReservedUntilTokenId) {
            revert CANNOT_DECREASE_RESERVE();
        }
++      // Remove all previous assigned `tokenRecipient` from founders

        // Set the new reserve
        reservedUntilTokenId = newReservedUntilTokenId;

++      // Assign the new `tokenRecipient` founders array using the new reservedUntilTokenId

        emit ReservedUntilTokenIDUpdated(newReservedUntilTokenId);
    }
```
