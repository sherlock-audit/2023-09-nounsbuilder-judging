Clean Purple Penguin

medium

# isContract modifier is not valid condition to check address

### Summary

The `isContract` function can be bypassed.

### Vulnerability Detail

The isContract function, widely employed in multiple contracts, utilizes code size as a criterion to discern whether an address represents a contract. However, this method can be circumvented, rendering it an unreliable requirement for checking any address.

### Impact

Address.isContract() is not a reliable way of checking if the input is an EOA

### Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/lib/token/ERC721.sol#L165

### Tool Used

Manual Review

### Recommendation

It is advisable to refrain from using the isContract function.

```solidity
if (ERC721TokenReceiver(_to).onERC721Received(msg.sender, _from, _tokenId, "") != ERC721TokenReceiver.onERC721Received.selector) revert INVALID_RECIPIENT();
```

In this if statement, the condition `Address.isContract(_to)` should be removed. Checking whether the address can receive ERC721 is sufficient.