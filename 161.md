Best Satin Otter

medium

# `Token._mintWithVesting()` distribution is not fair for all the founders

## Summary
- Tokens are not fairly distributed between the different founders.
- The final tokens distributed will not respect accurately, in most cases, the `ownershipPct` assigned to each founder.

## Vulnerability Detail
In the `Token` contract the [_addFounders()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L117-L182) function is called upon initialization to add founders and compute their vesting allocations. This is achieved by reserving a mapping of [0-100] token indices, such that if a new token mint ID % 100 is reserved, it's sent to the appropriate founder.

The `Auction` contract will call the `token.mint()` function every time an auction is created. This function internally calls the `_mintWithVesting()` function:
```solidity
/// @notice Mints tokens to the caller and handles founder vesting
function mint() external nonReentrant onlyAuctionOrMinter returns (uint256 tokenId) {
    tokenId = _mintWithVesting(msg.sender);
}
```

Let's suppose that each auction lasts 24 hours (`_auctionParams.duration` = 24 hours). This means that any possible vested tokens will only be minted every 24 hours.

Moreover let's assume the following founder parameters:
```solidity
/// @notice The founder parameters
/// @param wallet The wallet address
/// @param ownershipPct The percent ownership of the token
/// @param vestExpiry The timestamp that vesting expires
struct FounderParams {
    address wallet;
    uint256 ownershipPct;
    uint256 vestExpiry;
}
```

```solidity
IManager.FounderParams[] memory _founderParams = new IManager.FounderParams[](5);
_founderParams[0].wallet = founder1;
_founderParams[1].wallet = founder2;
_founderParams[2].wallet = founder3;
_founderParams[3].wallet = founder4;
_founderParams[4].wallet = founder5;
_founderParams[0].ownershipPct = 20;
_founderParams[1].ownershipPct = 16;
_founderParams[2].ownershipPct = 12;
_founderParams[3].ownershipPct = 8;
_founderParams[4].ownershipPct = 4;
_founderParams[0].vestExpiry = block.timestamp + 2 weeks;
_founderParams[1].vestExpiry = block.timestamp + 2 weeks;
_founderParams[2].vestExpiry = block.timestamp + 2 weeks;
_founderParams[3].vestExpiry = block.timestamp + 2 weeks;
_founderParams[4].vestExpiry = block.timestamp + 2 weeks;
```

After 2 weeks, or which is the same, after 20 different auctions, these are the token balances of each founder:
- After 2 weeks V:
- contract_Token.totalSupply() -> 36
- contract_Token.balanceOf(founder1) -> 7 -> 7/36 = 19.44% < 20% of ownershipPct assigned
- contract_Token.balanceOf(founder2) -> 6 -> 6/36 = 16.66% > 16% of ownershipPct assigned
- contract_Token.balanceOf(founder3) -> 4 -> 4/36 = 11.11% < 12% of ownershipPct assigned
- contract_Token.balanceOf(founder4) -> 3 -> 3/36 = 8.33% > 8% of ownershipPct assigned
- contract_Token.balanceOf(founder5) -> 2 -> 2/36 = 5.56% > 4% of ownershipPct assigned 
- 22 tokens of the total 36 minted were distributed between founders. 

These results are more or less accurate but now lets change the `vestExpiry` of each founder to `3 days` instead of `2 weeks` and repeat the test:
- After 3 days V:
- contract_Token.totalSupply() -> 10
- contract_Token.balanceOf(founder1) -> 2 -> 2/10 = 20% == 20% of ownershipPct assigned
- contract_Token.balanceOf(founder2) -> 2 -> 2/10 = 20% > 16% of ownershipPct assigned 
- contract_Token.balanceOf(founder3) -> 1 -> 1/10 = 10% < 12% of ownershipPct assigned
- contract_Token.balanceOf(founder4) -> 1 -> 1/10 = 10% > 8% of ownershipPct assigned
- contract_Token.balanceOf(founder5) -> 1 -> 1/10 = 10% > 4% of ownershipPct assigned
- 7 tokens of the total 10 minted were distributed between founders. 

The difference is quite important and totally unfair especially for `founder3`. On the other hand some founders were given way more ownership than what they were actually supposed to receive. See `founder5` example. Note that this scenario will likely happen and will be aggravated if the amount of founders is even higher.

## Impact
The final tokens distributed will not respect accurately, in most cases, the `ownershipPct` assigned to each founder.

## Code Snippet
Both tests can be found in [this gist](https://gist.github.com/r0bert-ethack/9d96da8361c81fcaa3539f72666a3b4b).

Run the 2 weeks test sample with:
```solidity
forge test -vv --match-contract UnfairDistribution --match-test test_checkDistribution1
```

Run the 3 days test sample with:
```solidity
forge test -vv --match-contract UnfairDistribution --match-test test_checkDistribution2
```

## Tool used
Manual Review

## Recommendation
It is recommended to redesign the `Token._addFounders()` function so it distributes accurately the tokens minted. 
