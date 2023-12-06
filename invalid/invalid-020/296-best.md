Round Chartreuse Yak

medium

# Use `SafeCast` when set founder's `vestExpiry`.

## Summary
It is not safe to cast `uint256` to `uint32` without using `SafeCast` when setting founder's `vestExpiry`. Founders may set a big expiry, expecting that they will get the vesting token all the time. If the expiry exceed `type(uint32).max`, then the cast is unexpected, which may result in a really small (or even 0) `vestExpiry`. In such case, founders will not get all the vesting tokens as they expect.

## Vulnerability Detail
The input `_founders[i].vestExpiry` (type is `uint256`) is casted to `uint32` directly, which is unsafe when `_founders[i].vestExpiry` is greater than `type(uint32).max`.
```solidity
// Function: Token.sol#_addFounders

153:                newFounder.vestExpiry = uint32(_founders[i].vestExpiry);
```
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L153

## Impact
With unsafe cast, founders may get a really small `vestExpiry`. As a result, they may not get all the vesting tokens as they expect.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L153

## Tool used

Manual Review

## Recommendation
Use `SafeCast.toUint32` when set `newFounder.vestExpiry`.