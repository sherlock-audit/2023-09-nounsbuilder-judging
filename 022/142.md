Basic Mauve Sealion

medium

# `initialize` funtion in `Manager` contract can be front-run due to lack of access control

## Summary

Anyone can call initialize function in `Manager` contract and gain ownership of this contract. This contract plays critical role as it is used to deploy and initialize other contracts (`Auction`, `Token` and other). What's more important is that the owner of the manager contract has ability to offer implementation upgrades for created DAOs.

## Vulnerability Detail

```solidity
    /// @notice Initializes ownership of the manager contract
    /// @param _newOwner The owner address to set (will be transferred to the Builder DAO once its deployed)
    function initialize(address _newOwner) external initializer {
        // Ensure an owner is specified
        if (_newOwner == address(0)) revert ADDRESS_ZERO();

        // Set the contract owner
        __Ownable_init(_newOwner);
    }
```

As we can see here there is no access control for `initialize` function. As mentioned before, owner has ability to offer implementation upgrades for created DAOs. When the `initialize` function is front-run and other user take control of the Manager contract he could never call `registerUpgrade`.

## Impact

`initialize` function can be front-run. Upgrading functionality can be bricked.

## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/manager/Manager.sol#L73-L79

## Tool used

Manual Review

## Recommendation

Implement proper access control in `Manager` contract so that only desired address (DAO address) could gain ownership of this contract.
