Skinny Frost Puppy

medium

# hardcoded gas in Auction contract may break the destination contracts logics if they don't support WETH

## Summary
in Auction contract when contract send ETH, it uses hard coded gas for the external call. the issue is that if the target contract get updated then 50K gas may not be enough to process its logics and code would revert. then Auction would try to send ETH as WETH to the target contract but code may not support it.
the outgoing ETH is used to send previous bidder ETH and treasury funds. in current implementation treasury doesn't handle the WETH balance and if code get updated and 50K gas was not enough for its logics then those WETH would stuck in the contract.
the issue can trigger by this events:
1- Treasury contract update.
2- 3rd party contract updates working with auction.
3- Ethereum gas profile cost upgrade.

## Vulnerability Detail
This is part of `_handleOutgoingTransfer()` code, which is called during new bid to send back previous bidder funds, and during auction settlements to send treasury funds:
```javascript
function _handleOutgoingTransfer(address _to, uint256 _amount) private {
        // Ensure the contract has enough ETH to transfer
        if (address(this).balance < _amount) revert INSOLVENT();

        // Used to store if the transfer succeeded
        bool success;

        assembly {
            // Transfer ETH to the recipient
            // Limit the call to 50,000 gas
            success := call(50000, _to, _amount, 0, 0, 0, 0)
        }

        // If the transfer failed:
        if (!success) {
            // Wrap as WETH
            IWETH(WETH).deposit{ value: _amount }();

            // Transfer WETH instead
            bool wethSuccess = IWETH(WETH).transfer(_to, _amount);
```
as you can see code uses hardcoded 50K gas to send ETH and if that transaction fails then code convert ETH to WETH and transfers the WETH.
the issue is that 50K gas may not be enough to process the receiving ETH in the call's target contract, while target contract couldn't handle the WETH balance.
for example in current implementation the treasury can't handle the WETH balance and if in future updates it requires more than 50K gas then Auction contract would send WETH to Treasury and those funds would stuck in the contract.
This can happen to any contract that works with Auction contract too. if they get updated while not processing WETH funds will stuck in them.
The issue will happen if there was some updates to Ethereum gas profile too. then current contracts that works with 50k gas will break and they would receive the WETH that they can't handle.

## Impact
funds can be locked as WETH in addresses that can't handle it.

## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L517-L538

## Tool used
Manual Review

## Recommendation
don't use hardcoded gas.