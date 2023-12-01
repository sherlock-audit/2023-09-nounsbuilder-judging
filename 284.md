Nutty Rouge Toad

medium

# Minting functions and auctions can be DOS'd due to running Out of Gas

See `Token.sol' code [here](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol)
See `Auction.sol::_createAuction' code [here](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L292-L329)
See `Token.sol::_isForFounder()` code [here](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/token/Token.sol#L263-L285)
See `Token.sol::mintWithVesting()` code [here](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/token/Token.sol#L230-L243)
See 


## Summary
`mintBatchTo()`, `mintTo()`, `mint()`, and hence `_createAuction()`,  may run out of gas if all founder percentages are allocated given the need for 100 loops and 100 mints per token minted. These functions are used to mint tokens as needs require, including when creating auctions.

## Vulnerability Detail
In order to find a token that can be minted; [`Token.sol::mintWithVesting()`](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/token/Token.sol#L230-L243) passes a token Id to [`Token.sol::_isForFounder()`](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/token/Token.sol#L263-L285) which loops through all token Ids searching for a token which can be minted. If it encounters any tokens which are assigned to a founder it mints that token for the founder and continues.
```solidity
    function _isForFounder(uint256 _tokenId) private returns (bool) {
        // Get the base token id
        uint256 baseTokenId = _tokenId % 100;

        // If there is no scheduled recipient:
        if (tokenRecipient[baseTokenId].wallet == address(0)) {
            return false;

            // Else if the founder is still vesting:
        } else if (block.timestamp < tokenRecipient[baseTokenId].vestExpiry) {
            // Mint the token to the founder
            _mint(tokenRecipient[baseTokenId].wallet, _tokenId);

            return true;

            // Else the founder has finished vesting:
        } else {
            // Remove them from future lookups
            delete tokenRecipient[baseTokenId];

            return false;
        }
    }
``` 
When all 99 founder token allocation is assigned; this it will cost about `5.5 million Gas` per token to mint tokens via `Token.sol::mintBatchTo()`, `Token.sol::mint()` and `Token.sol::mintTo()`.

## Impact
`mintBatchTo()`, `mintTo()` and `mint()`can be so expensive as to be unuseable.
This effectively DOS's their use and at the same time those functions which use them such as `createAuction()` and `settleAuction()`; DOSing the protocol's functionality. 
Given that This can severly restricting the protocol's ability to distribute tokens outside auctions and even a far lower amount of allocated tokens than used in the POC can make minting prohibitively expensive.

## Code Snippet
### Proof of Concept
Add the following tests to `Token.t.sol`:

<details>

<summary>`mintBatchTo()` POC</summary>
Minting six tokens using `mintBatchTo()` will cost `33_099_798` gas

```solidity
function test_MintBatchTo_OOG() public {
        // set percentage array
        uint256[] memory percentages = new uint256[](1);
            percentages[0] = 99;

        // set founder array
        address[] memory founders = new address[](1);
            founders[0] = founder;

        // set vesting array
        uint256[] memory vesting = new uint256[](1);
            vesting[0] = 2 days;

        uint256 reservedUntilTokenId = 0;

        // deploy with one founder holding 99%
        vm.prank(founder);
        deployWithCustomFounders(founders, percentages, vesting);
        
        vm.prank(founder);
        auction.unpause();

        // measure starting gas
        uint256 gasStart = gasleft();

        vm.prank(address(auction));
        uint256[] memory tokenIds = token.mintBatchTo(uint256(6), address(0x9));

        // measure ending gas
        uint256 gasEnd = gasleft();
        uint256 gasUsed = gasStart - gasEnd;

        // Assert gasUSed is greater than 30 million
        assertGt(gasUsed, 30_000_000); // 33_099_798
        console.log("GAS USED:", gasStart - gasEnd); 
}

```
</details>

<details>
<summary>`mint()` POC</summary>
Minting single tokens using `mint()` will cost `5_500_208` gas

```solidity
function test_Mint_OOG() public {
        // set percentage array
        uint256[] memory percentages = new uint256[](1);
            percentages[0] = 99;

        // set founder array
        address[] memory founders = new address[](1);
            founders[0] = founder;

        // set vesting array
        uint256[] memory vesting = new uint256[](1);
            vesting[0] = 2 days;

        uint256 reservedUntilTokenId = 0;

        // deploy with one founder holding 99%
        vm.prank(founder);
        deployWithCustomFounders(founders, percentages, vesting);
        
        vm.prank(founder);
        auction.unpause();

        // measure starting gas
        uint256 gasStart = gasleft();

        vm.prank(address(auction));
        uint256[] memory tokenIds = token.mintTo();

        // measure ending gas
        uint256 gasEnd = gasleft();
        uint256 gasUsed = gasStart - gasEnd;

        // Assert gasUSed is greater than 30 million
        assertGt(gasUsed, 5_000_000); // 5_500_208
        console.log("GAS USED:", gasStart - gasEnd); 
}

```


</details>

## Tool used
Foundry
Manual Review

## Recommendation
Add a storage variable which can be updated after minting and used as a starting point rather than looping each time from `reservedUntilTokenId` which effectively works as token 0.