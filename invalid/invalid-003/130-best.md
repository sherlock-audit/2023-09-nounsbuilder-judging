Fresh Obsidian Spider

medium

# Fixed gas limit when transferring ETH, could lead to possible loss of user funds

## Summary
The auction contract's _handleOutgoingTransfer function poses a risk due to its fixed gas limit of 50,000 for ETH transfers. This approach, while mitigating the 63/64th attack, can lead to unintended consequences, particularly when interacting with user contracts with complex logic in their receive() functions.

## Vulnerability Detail
The function uses a fixed gas limit for ETH transfers. If a user's contract has a `receive()` function that requires more than 50,000 gas, the transfer will fail and the contract will send WETH instead. This is risky on networks like Optimism, Base, Zora, or future Ethereum forks where gas costs for opcodes may change. If the user's contract cannot handle ERC20 tokens, any WETH sent will be irretrievably locked in their contract.

## Impact
This design flaw can result in the permanent loss of funds for users, especially if their contracts are not equipped to handle WETH. It also raises concerns about the contract's adaptability to future network changes where gas requirements may vary.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L524-L528

## Tool used
vscode

Manual Review

## Recommendation
If the initial ETH transfer fails, before defaulting to WETH, the contract could attempt a retry with a higher gas limit that is safe.
