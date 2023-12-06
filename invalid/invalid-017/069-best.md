Skinny Frost Puppy

medium

# function mintFromReserve() doesn't return the excess ETH values

## Summary
function `mintFromReserve()` mints tokens from reserve using a merkle proof and use pay the `pricePerToken + BUILDER_DAO_FEE` for each token. the issue is that code checks that `msg.value` is bigger than what users had to pay but doesn't return the excess ETH that users pays. because price of the tokens can be changes by DAO funder, users may pay more to be sure that their transaction won't revert but it will result in losing funds.

## Vulnerability Detail
This is part of the `mintFromReserve()` code:
```javascript
        // Check value sent
        if (msg.value < _getTotalFeesForMint(settings.pricePerToken, claimCount)) {
            revert INVALID_VALUE();
        }
```
as you can see code revert when `msg.value` is lower than what use supposed to pay and code doesn't make sure that `msg.value` equal to the fee of minting tokens, so it would be possible for user to pay more ETH and transaction won't revert. also contract doesn't return the excess funds user paid and transfer them to the DAO treasury in `_distributeFees()`:
```javascript
    function _distributeFees(address tokenContract, uint256 quantity) internal {
        uint256 builderFee = quantity * BUILDER_DAO_FEE;
        uint256 value = msg.value;

        (, , address treasury, ) = manager.getAddresses(tokenContract);
        address builderRecipient = manager.builderRewardsRecipient();

        // Pay out fees to the Builder DAO
        protocolRewards.deposit{ value: builderFee }(builderRecipient, hex"00", "");

        // Pay out remaining funds to the treasury
        if (value > builderFee) {
            (bool treasurySuccess, ) = treasury.call{ value: value - builderFee }("");

            // Revert if treasury cannot accept funds
            if (!treasurySuccess) {
                revert TRANSFER_FAILED();
            }
        }
    }
```

because the price of the NFT can be changed by DAO funders by calling `setMintSettings()`, so it's a valid assumption that in some situations(for example DAO changing the price based on some market) users have to pay more ETH to make sure their transaction won't revert.

## Impact
users would lose funds when they send more ETH

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/minters/MerkleReserveMinter.sol#L189-L209
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/minters/MerkleReserveMinter.sol#L149-L152

## Tool used
Manual Review

## Recommendation
either return the extra ETH send by user or make suer `msg.value` equals to the total payment required.