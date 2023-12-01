Quiet Snowy Frog

high

# Underflow in the voting weight of the user

## Summary

The issue is turned on whenever the `balances[ ]` in `ERC721.sol` get updated so in `transferFrom()` , `_mint()` and `_burn()`
due to the wrong update of the voting weight in case the users are delegating 
## Vulnerability Detail

The user Y can delegate voting weight to an account X by invoking `ERC721Votes.delegate()
at this time the new voting weight of Y is 0 and X is 1

Now, if Y call `ERC721.transferFrom()` to transfer his token to recipient Z
the hood in `ERC721Votes._afterTokenTransfer()`

```solidity
File: ERC721Votes.sol

312:     function _afterTokenTransfer(
313:         address _from,
314:         address _to,
315:         uint256 _tokenId
316:     ) internal override {
317:         // Transfer 1 vote from the sender to the recipient
318:         _moveDelegateVotes(delegates(_from), delegates(_to), 1);
319: 
320:         super._afterTokenTransfer(_from, _to, _tokenId);
321:     }

```
Because the voting weight of Y is 0 and `_moveDelegateVotes()` has an unchecked block inside it 
the computation in this line will get underflow

```solidity
File: ERC721Votes.sol

233: // Update their voting weight
234: _writeCheckpoint(_from, newCheckpointId, prevCheckpointId, prevTimestamp, prevTotalVotes, prevTotalVotes - _amount);
```
and the `_writeCheckpoint()` will update the records `checkpoint.votes` to the max value of `uint192`

```solidity
File: ERC721Votes.sol

296:   // Store the new voting weight and the current time
297:   checkpoint.votes = uint192(_newTotalVotes);

```
NB: 
The same issue if `Minter` call `ERC721Votes.delegate()` and then invoke `Token.burn()`
and the same with this scenario 

1.  we have user_01 and user_02
2.  user_01 has 2 NFTs and 2  voting weight
3.  user_02 has 0 NFTs and 0  voting weight

- user_01 call `ERC721Votes.delegate()` to the user_02
So:
  - user_01 has 2 NFTs and 0  voting weight
  - user_02 has 0 NFTs and 2  voting weight

- user_01 has won a new auction
So:
  - user_01 has 3 NFTs and 1  voting weight
  - user_02 has 0 NFTs and 2  voting weight

Now:
- user_01 call `ERC721Votes.delegate()` to the `address(0)` (self-delegate)
This line `_moveDelegateVotes(prevDelegate, _to, balanceOf(_from))` will try to move 3  voting weight from user_02 which only has 2  voting weight

## Impact

 - underflow in the voting weight of the user so he can manipulate any government proposal or create a malicious proposal and vote for it
 
## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/token/ERC721.sol#L235-L239

## Tool used

Manual Review

## Recommendation

Use `_beforeTokenTransfer()` to check if `from` or `to` delegate their voting weight to another address 
if yes you need to deal with this case correctly  
