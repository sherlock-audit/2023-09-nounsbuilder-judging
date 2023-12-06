Proud Orange Blackbird

high

# The migration of DAO from L1 to L2 is not performed correctly by L2MigrationDeployer.

## Summary
Users want to migrate their `DAO` from `L1` to `L2` using `L2MigrationDeployer`, but encounter difficulties in achieving successful migration. 
This may result in the `DAO` not functioning properly or behaving differently on `L2` compared to `L1`.
## Vulnerability Detail
Let me illustrate with an example:
Consider a `DAO` on `L1` that has been working for a certain period.
The `DAO` has the following parameters: `reservedUntilTokenId` is set to 50, and there are two `founders` with percentage allocations of `2` and `3`.
From these parameters, the following facts become apparent:
```solidity
tokenRecipient[50] = tokenRecipient[0] = founder1 (schedule = 100 / 2 = 50)
tokenRecipient[51] = tokenRecipient[84] = tokenRecipient[17] = founder2 (schedule = 100 / 3 = 33)
```
In the first auction, token `52` was created, with token `50` assigned to `founder1` and token `51` assigned to `founder2`.
Some subsequent auctions achieved success, while others failed, leading to the burning of certain tokens, including token `53`.
Let's assume that `settings.mintCount` has been increased to `10`.
The `DAO` owners wish to migrate to `L2`, successfully deploying to `L2` using `L2MigrationDeployer`. 
Token holders on `L1` claimed their tokens on `L2` via `MerkleReserveMinter` minter, and two scenarios arise:

- `Scenario 1`: Seeding `50` as `reservedUntilTokenId`.

Following the migration, the `DAO` initiated an auction on `L2`.
To proceed with an auction, a new token must be created. The `mint` function is invoked,
```solidity
function mint() external nonReentrant onlyAuctionOrMinter returns (uint256 tokenId) {
    tokenId = _mintWithVesting(msg.sender);
}
```
In the `_mintWithVesting` function, the first `tokenId` is `50` (due to `reservedUntilTokenId` being set to `50` and `settings.mintCount` being 0 for the new `DAO` on `L2`).
```solidity
function _mintWithVesting(address recipient) internal returns (uint256 tokenId) {
      unchecked {
          do {
              tokenId = reservedUntilTokenId + settings.mintCount++;
          } while (_isForFounder(tokenId));
      }
      _mint(recipient, tokenId);
  }
```
```solidity
function _isForFounder(uint256 _tokenId) private returns (bool) {
    uint256 baseTokenId = _tokenId % 100;
    if (tokenRecipient[baseTokenId].wallet == address(0)) {
        return false;
    } else if (block.timestamp < tokenRecipient[baseTokenId].vestExpiry) {
        _mint(tokenRecipient[baseTokenId].wallet, _tokenId);
        return true;
    } else {
        delete tokenRecipient[baseTokenId];
        return false;
    }
}
```
For token `50`, if `founder1`'s vesting period has not expired, it should be assigned to `founder1`. 
Otherwise, it should be assigned to the `caller` (`auction` contract in this case).
In both cases, the `_mint` function should be called.
```solidity
function _mint(address _to, uint256 _tokenId) internal override {
     super._mint(_to, _tokenId);
}
```
This invoked `ERC721`'s `_mint` function, results in a transaction revert because token `50` was already claimed by `founder1` using `MerkleReserveMinter`.
```solidity
function _mint(address _to, uint256 _tokenId) internal virtual {
    if (_to == address(0)) revert ADDRESS_ZERO();
    if (owners[_tokenId] != address(0)) revert ALREADY_MINTED();
}
```
The DAO won't function properly.


- `Scenario 2`: Seeding `60` as `reservedUntilTokenId`. 

Here `60` is the sum of `50` (`reservedUntilTokenId` on `L1`) and `10` (`settings.mintCount` on `L1`).
Then the values of `tokenRecipients` will differ between `L1` and `L2` because of the differences in `reservedUntilTokenId` between the two.
Additionally, some tokens that were burnt on `L1` can now be claimed on `L2`.
For instance, consider token `53`, which was burnt on `L1` and couldn't be assigned to any user.
Because there is only one function to claim this token.
```solidity
function mintFromReserveTo(address recipient, uint256 tokenId) external nonReentrant onlyMinter {
    if (tokenId >= reservedUntilTokenId) revert TOKEN_NOT_RESERVED();
    _mint(recipient, tokenId);
}
```
On `L1`, this `ID` is greater than the `reservedUntilTokenId` (`50`), leading to a revert. 
However, on `L2`, this `ID` can be successfully claimed because the `reservedUntilTokenId` (`60`) is greater than this `ID`.
Furthermore, it is possible to adjust the `reservedUntilTokenId` on `L2` because of `settings.mintCount` is still `0`.
```solidity
function setReservedUntilTokenId(uint256 newReservedUntilTokenId) external onlyOwner {
    if (settings.mintCount > 0) {
        revert CANNOT_CHANGE_RESERVE();
    }
}
```
## Impact
Migrating a functioning `DAO` from `L1` to a `DAO` with identical parameters and balances on `L2` can be challenging. 
However, a solution can be implemented quite easily. 
Please review the following recommendation:
## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L201-L203
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L230-L243
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L263-L285
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L248-L259
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/token/ERC721.sol#L191-L207

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L211-L217
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L486-L503
## Tool used

Manual Review
## Recommendation
Please add below function to `token`.
```solidity
function setSettingsMintCounter(uint88 _mintCounter) external onlyOwner {
    if (settings.mintCount != 0) revert ALREADY_CHANGED();

    settings.mintCount = _mintCounter;
}
```
Additionally, make a slight modification to the `deploy` function on `L2MigrationDeployer`.
```solidity
function deploy(
    IManager.FounderParams[] calldata _founderParams,
    IManager.TokenParams calldata _tokenParams,
    IManager.AuctionParams calldata _auctionParams,
    IManager.GovParams calldata _govParams,
    MerkleReserveMinter.MerkleMinterSettings calldata _minterParams,
    + uint88 _mintCounter,
) external returns (address token) {
    IToken(_token).updateMinters(minters);
    + IToken(_token).setSettingsMintCounter(_mintCounter);
}
```
By following this approach, we can seamlessly migrate the `DAO` from `L1` to an identical `DAO` on `L2`.

