Clean Purple Penguin

medium

# Unsafe transferOwnership

## Summary

The contract utilizes Ownable.sol for ownership management. But its not safe.

## Vulnerability Detail

Ownable2Step offers a safer alternative to Ownable for smart contracts by preventing accidental transfer of ownership to a mistyped address. Unlike the direct transfer in Ownable, Ownable2Step completes the transfer only when the new owner explicitly accepts ownership. Released in January 2023 with the OpenZeppelin version 4.8 update, its documentation and source code are available.

## Impact

### Scenario: Mistaken Ownership Transfer

#### Context:
Suppose there is a decentralized application (DApp) that utilizes a smart contract for managing a critical function, such as a decentralized auction. The smart contract employs the Ownable pattern from OpenZeppelin for ownership management.

#### Initial Implementation:
Initially, the DApp uses the standard Ownable implementation:

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";

contract Auction is Ownable {
    // ... contract logic ...
}
```

#### Vulnerability:
In a rush to update the contract or due to a simple human error, the contract owner mistakenly calls the `transferOwnership` function with a mistyped or non-existent address:

```solidity
function updateOwner(address _newOwner) external onlyOwner {
    transferOwnership(_newOwner); // Mistakenly using a mistyped or non-existent address
}
```

In the standard Ownable implementation, this operation would succeed without any further confirmation or validation.

#### Consequences:
The unintended ownership transfer could lead to several undesirable outcomes:

1. **Loss of Control:** The ownership of the smart contract is transferred to an unintended address, potentially compromising the control and management of the decentralized auction.

2. **Security Risks:** The new owner, if not a trusted party, may exploit the ownership privileges to manipulate the contract, disrupt the auction, or drain funds.

3. **Recovery Challenges:** Rectifying the situation may be challenging, especially if the new owner is uncooperative or if the mistyped address does not exist.

## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L27

## Tool Used

Manual Review

## Recommendation

To enhance contract security, it is advised to implement and utilize Ownable2Step to mitigate potential vulnerabilities, as demonstrated in the provided example.