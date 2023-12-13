# Issue H-1: when reservedUntilTokenId > 100 first funder loss 1% NFT 

Source: https://github.com/sherlock-audit/2023-09-nounsbuilder-judging/issues/42 

## Found by 
0x52, 0xReiAyanami, 0xbepresent, 0xcrunch, 0xmystery, 0xpep7, Aamirusmani1552, Ch\_301, Falconhoof, HHK, Jiamin, Juntao, KingNFT, Kow, Krace, KupiaSec, Nyx, SilentDefendersOfDeFi, SovaSlava, almurhasan, ast3ros, bin2chen, cawfree, chaduke, circlelooper, coffiasd, dany.armstrong90, deepkin, dimulski, ge6a, ggg\_ttt\_hhh, giraffe, gqrp, pontifex, qpzm, rvierdiiev, saian, unforgiven, whoismxuse, xAriextz, ydlee, zraxx
## Summary
The incorrect use of `baseTokenId = reservedUntilTokenId` may result in the first `tokenRecipient[]` being invalid, thus preventing the founder from obtaining this portion of the NFT.

## Vulnerability Detail

The current protocol adds a parameter `reservedUntilTokenId` for reserving `Token`.
This parameter will be used as the starting `baseTokenId` during initialization.

```solidity
    function _addFounders(IManager.FounderParams[] calldata _founders, uint256 reservedUntilTokenId) internal {
...

                // Used to store the base token id the founder will recieve
@>              uint256 baseTokenId = reservedUntilTokenId;

                // For each token to vest:
                for (uint256 j; j < founderPct; ++j) {
                    // Get the available token id
                    baseTokenId = _getNextTokenId(baseTokenId);

                    // Store the founder as the recipient
                    tokenRecipient[baseTokenId] = newFounder;

                    emit MintScheduled(baseTokenId, founderId, newFounder);

                    // Update the base token id
                    baseTokenId = (baseTokenId + schedule) % 100;
                }
            }
..

    function _getNextTokenId(uint256 _tokenId) internal view returns (uint256) {
        unchecked {
@>          while (tokenRecipient[_tokenId].wallet != address(0)) {
                _tokenId = (++_tokenId) % 100;
            }

            return _tokenId;
        }
    }
```

Because `baseTokenId = reservedUntilTokenId` is used, if `reservedUntilTokenId>100`, for example, reservedUntilTokenId=200, the first `_getNextTokenId(200)` will return `baseTokenId=200 ,  tokenRecipient[200]=newFounder`.

Example:
reservedUntilTokenId = 200
founder[0].founderPct = 10

In this way, the `tokenRecipient[]` of `founder` will become
tokenRecipient[200].wallet = founder   ( first will call _getNextTokenId(200) return 200)
tokenRecipient[10].wallet = founder      ( second will call _getNextTokenId((200 + 10) %100 = 10) )
tokenRecipient[20].wallet = founder
...
tokenRecipient[90].wallet = founder


However, this `tokenRecipient[200]` will never be used, because in `_isForFounder()`, it will be modulo, so only `baseTokenId < 100` is valid. In this way, the first founder can actually only `9%` of NFT.

```solidity
    function _isForFounder(uint256 _tokenId) private returns (bool) {
        // Get the base token id
@>      uint256 baseTokenId = _tokenId % 100;

        // If there is no scheduled recipient:
        if (tokenRecipient[baseTokenId].wallet == address(0)) {
            return false;

            // Else if the founder is still vesting:
        } else if (block.timestamp < tokenRecipient[baseTokenId].vestExpiry) {
            // Mint the token to the founder
@>          _mint(tokenRecipient[baseTokenId].wallet, _tokenId);

            return true;

            // Else the founder has finished vesting:
        } else {
            // Remove them from future lookups
            delete tokenRecipient[baseTokenId];

            return false;
        }
    }
```

## POC

The following test demonstrates that `tokenRecipient[200]` is for founder.

1. need change tokenRecipient to public , so can assertEq
```diff
contract TokenStorageV1 is TokenTypesV1 {
    /// @notice The token settings
    Settings internal settings;

    /// @notice The vesting details of a founder
    /// @dev Founder id => Founder
    mapping(uint256 => Founder) internal founder;

    /// @notice The recipient of a token
    /// @dev ERC-721 token id => Founder
-   mapping(uint256 => Founder) internal tokenRecipient;
+   mapping(uint256 => Founder) public tokenRecipient;
}
```

2. add to `token.t.sol`
```solidity
    function test_lossFirst(address _minter, uint256 _reservedUntilTokenId, uint256 _tokenId) public {
        deployAltMock(200);
        (address wallet ,,)= token.tokenRecipient(200);
        assertEq(wallet,founder);
    }
```

```console
$ forge test -vvv --match-test test_lossFirst

Running 1 test for test/Token.t.sol:TokenTest
[PASS] test_lossFirst(address,uint256,uint256) (runs: 256, μ: 3221578, ~: 3221578)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 355.45ms
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

## Impact

when reservedUntilTokenId > 100 first funder loss 1% NFT

## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L161

## Tool used

Manual Review

## Recommendation
1. A better is that the baseTokenId always starts from 0.
```diff
    function _addFounders(IManager.FounderParams[] calldata _founders, uint256 reservedUntilTokenId) internal {
...

                // Used to store the base token id the founder will recieve
-               uint256 baseTokenId = reservedUntilTokenId;
+               uint256 baseTokenId =0;
```
or

2. use `uint256 baseTokenId = reservedUntilTokenId  % 100;`
```diff
    function _addFounders(IManager.FounderParams[] calldata _founders, uint256 reservedUntilTokenId) internal {
...

                // Used to store the base token id the founder will recieve
-               uint256 baseTokenId = reservedUntilTokenId;
+               uint256 baseTokenId = reservedUntilTokenId  % 100;
```



## Discussion

**neokry**

This is valid and is the core issue behind #247 as well. baseTokenId should start at 0 in `addFounders`

**nevillehuang**

I initially separated the 4 findings below, but I agree, #177, #247 and #67 are only possible because of the following lines of code [here](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L161), wherein `_addFounder()`, `baseTokenId` is incorrectly initialized to `reservedUntilTokenId ` in `addFounders()`, which is the root cause of the issue, and once fixed, all the issues will be fixed too. There are 4 impacts mentioned by watsons. 

```solidity
uint256 baseTokenId = reservedUntilTokenId;
```

1. Previous founders that are meant to be deleted are retained causing them to continue receiving minted NFTs --> High severity, since it is a definite loss of funds

2. #247: Any `reserveTokenId` greater than 100 will cause a 1% loss of NFT for founder --> High severity, since it is a definite loss of funds for founder as long as `reservedUntilTokenId ` is set greater than 100, which is not unlikely

3. #177: This is essentially only an issue as `baseTokenId` is incorrectly set as `reservedUntilTokenId` but will cause a definite loss of founders NFT if performed, so keeping as duplicate

4. #67: This is closely related to the above finding (177), where a new update to `reservedUntilTokenId` via `setReservedUntilTokenId` can cause over/underallocation NFTs so keeping as duplicate


However, in the context of the audit period, I could also see why watsons separated these issues, so happy to hear from watsons during escalation period revolving deduplication of these issues.

# Issue H-2: Adversary can permanently brick auctions due to precision error in Auction#_computeTotalRewards 

Source: https://github.com/sherlock-audit/2023-09-nounsbuilder-judging/issues/251 

## Found by 
0x52, Bauer, Brenzee, HHK, Kow, KupiaSec, SilentDefendersOfDeFi, coffiasd, cu5t0mPe0, dany.armstrong90, ggg\_ttt\_hhh, gqrp, pontifex, unforgiven, xAriextz
## Summary

When batch depositing to ProtocolRewards, the msg.value is expected to match the sum of the amounts array EXACTLY. The issue is that due to precision loss in Auction#_computeTotalRewards this call can be engineered to always revert which completely bricks the auction process.

## Vulnerability Detail

[ProtocolRewards.sol#L55-L65](https://github.com/ourzora/zora-protocol/blob/8d1fe9bdd79a552a8f74b4712451185f6aebf9a0/packages/protocol-rewards/src/ProtocolRewards.sol#L55-L65)

        for (uint256 i; i < numRecipients; ) {
            expectedTotalValue += amounts[i];

            unchecked {
                ++i;
            }
        }

        if (msg.value != expectedTotalValue) {
            revert INVALID_DEPOSIT();
        }

When making a batch deposit the above method is called. As seen, the call with revert if the sum of amounts does not EXACTLY equal the msg.value.

[Auction.sol#L474-L507](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L474-L507)

        uint256 totalBPS = _founderRewardBps + referralRewardsBPS + builderRewardsBPS;

        ...

        // Calulate total rewards
        split.totalRewards = (_finalBidAmount * totalBPS) / BPS_PER_100_PERCENT;

        ...

        // Initialize arrays
        split.recipients = new address[](arraySize);
        split.amounts = new uint256[](arraySize);
        split.reasons = new bytes4[](arraySize);

        // Set builder reward
        split.recipients[0] = builderRecipient;
        split.amounts[0] = (_finalBidAmount * builderRewardsBPS) / BPS_PER_100_PERCENT;

        // Set referral reward
        split.recipients[1] = _currentBidRefferal != address(0) ? _currentBidRefferal : builderRecipient;
        split.amounts[1] = (_finalBidAmount * referralRewardsBPS) / BPS_PER_100_PERCENT;

        // Set founder reward if enabled
        if (hasFounderReward) {
            split.recipients[2] = founderReward.recipient;
            split.amounts[2] = (_finalBidAmount * _founderRewardBps) / BPS_PER_100_PERCENT;
        }

The sum of the percentages are used to determine the totalRewards. Meanwhile, the amounts are determined using the broken out percentages of each. This leads to unequal precision loss, which can cause totalRewards to be off by a single wei which cause the batch deposit to revert and the auction to be bricked. Take the following example:

Assume a referral reward of 5% (500) and a builder reward of 5% (500) for a total of 10% (1000). To brick the contract the adversary can engineer their bid with specific final digits. In this example, take a bid ending in 19. 

    split.totalRewards = (19 * 1,000) / 100,000 = 190,000 / 100,000 = 1

    split.amounts[0] = (19 * 500) / 100,000 = 95,000 / 100,000 = 0
    split.amounts[1] = (19 * 500) / 100,000 = 95,000 / 100,000 = 0

Here we can see that the sum of amounts is not equal to totalRewards and the batch deposit will revert. 

[Auction.sol#L270-L273](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L270-L273)

    if (split.totalRewards != 0) {
        // Deposit rewards
        rewardsManager.depositBatch{ value: split.totalRewards }(split.recipients, split.amounts, split.reasons, "");
    }

The depositBatch call is placed in the very important _settleAuction function. This results in auctions that are permanently broken and can never be settled.

## Impact

Auctions are completely bricked

## Code Snippet

[Auction.sol#L244-L289](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L244-L289)

## Tool used

Manual Review

## Recommendation

Instead of setting totalRewards with the sum of the percentages, increment it by each fee calculated. This way they will always match no matter what.



## Discussion

**nevillehuang**

Contrary to #103 which only affects bidding, this causes a complete DoS of settlement of auctions, forcing the DAO to possibly have to redeploy to resolve the issue and continue auctions, so I believe high severity is fair.

# Issue M-1: MerkleReserveMinter minting methodology is incompatible with current governance structure and can lead to migrated DAOs being hijacked immediately 

Source: https://github.com/sherlock-audit/2023-09-nounsbuilder-judging/issues/249 

## Found by 
0x52, SilentDefendersOfDeFi, jerseyjoewalcott, nirohgo
## Summary

MerkleReserveMinter allows large number of tokens to be minted instantaneously which is incompatible with the current governance structure which relies on tokens being minted individually and time locked after minting by the auction. By minting and creating a proposal in the same block a user is able to create a proposal with significantly lower quorum than expected. This could easily be used to hijack the migrated DAO.

## Vulnerability Detail

[MerkleReserveMinter.sol#L154-L167](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L154-L167)

    unchecked {
        for (uint256 i = 0; i < claimCount; ++i) {
            // Load claim in memory
            MerkleClaim memory claim = claims[I];

            // Requires one proof per tokenId to handle cases where users want to partially claim
            if (!MerkleProof.verify(claim.merkleProof, settings.merkleRoot, keccak256(abi.encode(claim.mintTo, claim.tokenId)))) {
                revert INVALID_MERKLE_PROOF(claim.mintTo, claim.merkleProof, settings.merkleRoot);
            }

            // Only allowing reserved tokens to be minted for this strategy
            IToken(tokenContract).mintFromReserveTo(claim.mintTo, claim.tokenId);
        }
    }

When minting from the claim merkle tree, a user is able to mint as many tokens as they want in a single transaction. This means in a single transaction, the supply of the token can increase very dramatically. Now we'll take a look at the governor contract as to why this is such an issue.

[Governor.sol#L184-L192](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/governance/governor/Governor.sol#L184-L192)

        // Store the proposal data
        proposal.voteStart = SafeCast.toUint32(snapshot);
        proposal.voteEnd = SafeCast.toUint32(deadline);
        proposal.proposalThreshold = SafeCast.toUint32(currentProposalThreshold);
        proposal.quorumVotes = SafeCast.toUint32(quorum());
        proposal.proposer = msg.sender;
        proposal.timeCreated = SafeCast.toUint32(block.timestamp);

        emit ProposalCreated(proposalId, _targets, _values, _calldatas, _description, descriptionHash, proposal);

[Governor.sol#L495-L499](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/governance/governor/Governor.sol#L495-L499)

    function quorum() public view returns (uint256) {
        unchecked {
            return (settings.token.totalSupply() * settings.quorumThresholdBps) / BPS_PER_100_PERCENT;
        }
    }

When creating a proposal, we see that it uses a snapshot of the CURRENT total supply. This is what leads to the issue. The setup is fairly straightforward and occurs all in a single transaction:

1) Create a malicious proposal (which snapshots current supply)
2) Mint all the tokens
3) Vote on malicious proposal with all minted tokens

The reason this works is because the quorum is based on the supply before the mint while votes are considered after the mint, allowing significant manipulation of the quorum.

## Impact

DOA can be completely hijacked

## Code Snippet

[MerkleReserveMinter.sol#L129-L173](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/minters/MerkleReserveMinter.sol#L129-L173)

## Tool used

Manual Review

## Recommendation

Token should be changed to use a checkpoint based total supply, similar to how balances are handled. Quorum should be based on that instead of the current supply.



## Discussion

**neokry**

This is a valid issue but makes a big assumption that a malicious user is included in the merkle tree with a significant share of reserved tokens and the DAO has no veto set. This might be too much of an edge case to make the recommended changes to the token and governor contracts.

# Issue M-2: createBidWithReferral: bidder can steal builder's reward by setting refferal as themselves 

Source: https://github.com/sherlock-audit/2023-09-nounsbuilder-judging/issues/281 

## Found by 
0xReiAyanami, 0xmystery, Chinmay, EVDoc, HChang26, KupiaSec, Stoicov, deepkin, ggg\_ttt\_hhh, jasonxiale, klaus, n1punp, slvDev
## Summary

If a bidder sets their referrer to themselves, they can receive a portion of the rewards that would otherwise belong to the builder.

## Vulnerability Detail

 

When setting the `currentBidReferral` in the `createBidWithReferral` function, it does not check if `_referral` is a bidder. So the bidder can set themselves as a referrer.

```solidity
function createBidWithReferral(uint256 _tokenId, address _referral) external payable nonReentrant {
@>  currentBidReferral = _referral;
    _createBid(_tokenId);
}
```

Normally, if there are no referrals, the referral share of the reward should go to the builder. This means that the bidder can steal some of the builder's reward tokens.

```solidity
function _computeTotalRewards(
    address _currentBidRefferal,
    uint256 _finalBidAmount,
    uint256 _founderRewardBps
) internal view returns (RewardSplits memory split) {

    ...

    // Set referral reward
@>  split.recipients[1] = _currentBidRefferal != address(0) ? _currentBidRefferal : builderRecipient;
    split.amounts[1] = (_finalBidAmount * referralRewardsBPS) / BPS_PER_100_PERCENT;

    ...
}
```

Here's the PoC code. You can add it to Auction.t.sol and run it.

```solidity
function test_FounderBuilderAndReferralReward_Selfref() external {
    // Setup
    deployAltMock(founder, 500);

    vm.prank(manager.owner());
    manager.registerUpgrade(auctionImpl, address(rewardImpl));

    vm.prank(auction.owner());
    auction.upgradeTo(address(rewardImpl));

    vm.prank(founder);
    auction.unpause();

    // Check reward values

    vm.prank(bidder1);
    auction.createBidWithReferral{ value: 1 ether }(2, bidder1);
    vm.warp(10 minutes + 1 seconds);

    auction.settleCurrentAndCreateNewAuction();

    assertEq(token.ownerOf(2), bidder1);
    assertEq(token.getVotes(bidder1), 1);

    assertEq(address(treasury).balance, 0.88 ether);
    assertEq(address(rewards).balance, 0.03 ether + 0.04 ether + 0.05 ether);

    assertEq(MockProtocolRewards(rewards).balanceOf(zoraDAO), 0.03 ether);
    assertEq(MockProtocolRewards(rewards).balanceOf(bidder1), 0.04 ether); // bidder get reward
    assertEq(MockProtocolRewards(rewards).balanceOf(founder), 0.05 ether);
}
```

## Impact

Some of the builder's rewards will be stolen.

## Code Snippet

[https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L145](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L145)

[https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L500](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/db232c649b425c36f5a93607c95cfdf0e5962b2f/nouns-protocol/src/auction/Auction.sol#L500)

## Tool used

Manual Review

## Recommendation

1. Do not allow self-referrals.
2. Restrict that only the NFT owner(members of the DAO) can be set as the refferal to prevent bypassing using a user's other accounts. (If the user is already in DAO, then it could be bypassed. But this is actually means “referrer” of bid, so I think it's fine.)



## Discussion

**neokry**

This is expected behavior. Even if we restricted referrals to non biders its pretty easy for a bidder to set this value as another account or contract they own. We want to keep referrals open and if a handful of users are setting themselves as referrals we think this is a fair tradeoff vs the alternative of whitelisting a specific set of clients or users to claim referral rewards.

**nevillehuang**

While I agree there could be no potential fix for this issue, I think it is fair for watsons to bring this up given the default referral rewards are passed on to builders if there is no referrals. 

So this issue essentially incentivizes the users in DAO where builder rewards are set to self provide a "discount" and reduce rewards for buildersDAO, potentially defeating the intention of a referral system in the first place.

I believe this tradeoff could have been explicitly mention in the contest READ.ME as a known design decision, but since it is not, I am maintaining as medium severity.

**neokry**

Since this tradeoff was not mentioned in the readme or the spec we can mark this as a valid issue + wont fix

**nevillehuang**

#4, #213 and #283 mentions a different impact of bypassing minimum bid increments, but ultimately still fall under the same root cause of providing a self "discount" by setting referral to themselves/bidder controlled address. 

