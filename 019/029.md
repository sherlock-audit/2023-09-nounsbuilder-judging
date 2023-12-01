Clean Purple Penguin

medium

# No gap implementation can cause storage collusions for upgradable contracts

## Summary

Auction.sol has a problem – it's missing a crucial storage gap. This gap is essential for preventing storage conflicts during upgrades when adding more features.

## Vulnerability Detail

Upgradeable contracts, like Auction.sol, need a storage gap. This gap allows developers to smoothly add new features without causing issues with existing setups (as explained by OpenZeppelin). Without it, creating new code becomes tricky. The risk is that without this gap, the upgraded base contract might accidentally overwrite variables in the child contract when new elements are added. This could lead to unintended problems, like losing user funds or the entire contract breaking.

For more detailed information, check out this article: [OpenZeppelin Upgrades Plugins Documentation](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable).

## Impact

**Scenario: Trouble Without a Gap**

In the Nouns Protocol world, the excitement of enhancing Auction.sol for better auctions is dimmed by a small but crucial omission – the lack of a "storage gap."

**The Oops Moment:**

The upgraded Auction.sol launches without the storage gap. New features are introduced, and things seem positive.

**The Clash:**

Without the gap, new and existing features collide like puzzle pieces forced together.

**The Result:**

Client trust is compromised.

## Code Snippet

[Link to the vulnerable code in Auction.sol](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L27)

## Tool Used

Manual Review

## Recommendation

It is strongly advised to include an appropriate storage gap at the end of upgradeable contracts, as shown below. Developers are encouraged to refer to OpenZeppelin's upgradeable contract templates for guidance:

```solidity
uint256[50] private __gap;
```