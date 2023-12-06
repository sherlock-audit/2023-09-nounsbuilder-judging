Modern Jade Elephant

medium

# Revoked operator can still call the transferFrom function

## Summary
There is no way to revoke the approval which given via the `approve` function. They are able to call the `transferFrom` even after the tokenHolder revokes all permissions using the `setApprovalForAll` function. 
## Vulnerability Detail
Alice holds the token.

She approves Bob via `setApprovalForAll(_Bob, _true)`, which gives Bob access to the token, he's now operatorApproved.
```solidity
    function setApprovalForAll(address _operator, bool _approved) external {
        operatorApprovals[msg.sender][_operator] = _approved;

        emit ApprovalForAll(msg.sender, _operator, _approved);
    }
```
Bob approves himself `approve(_Bob, _tokenId)` and is now both operatorApproved and tokenApproved, for the tokens.
```solidity
    function approve(address _to, uint256 _tokenId) external {
        address owner = owners[_tokenId];

        if (msg.sender != owner && !operatorApprovals[owner][msg.sender]) revert INVALID_APPROVAL();

        tokenApprovals[_tokenId] = _to;

        emit Approval(owner, _to, _tokenId);
    }
```
Alice revokes Bob's approval by calling `setApprovalForAll(_Bob, _false)`, now Bob is no longer operatorApproved, but still tokenApproved.

Now Bob can still call the .transferFrom(_Alice, _to, _tokenId) which is subjected to call only by approved address.
```solidity
       if (msg.sender != _from && !operatorApprovals[_from][msg.sender] && msg.sender != tokenApprovals[_tokenId]) revert INVALID_APPROVAL();
```
The transfer will be successful.

## Impact
Bob can call the `transferFrom` and potentially cost Alice her tokens.
## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/lib/token/ERC721.sol#L115
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/lib/token/ERC721.sol#L102
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/lib/token/ERC721.sol#L125C1-L151C6
## Tool used

Manual Review

## Recommendation
Revoke all operator's tokenApprovals after his approval gets revoked or prevent self approvals by operators.