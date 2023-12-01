Bald Carob Swallow

high

# Token.sol#setReservedUntilTokenId: After changing reservedUntilTokenId, calling to updateFounders function does not delete all of tokenRecipient variables.

## Summary
In `Token.sol#_addFounders` function, `reservedUntilTokenId` is used to initialize `baseTokenId` variable as follows.
```solidity
File: Token.sol
120:     function _addFounders(IManager.FounderParams[] calldata _founders, uint256 reservedUntilTokenId) internal {
121:         // Used to store the total percent ownership among the founders
122:         uint256 totalOwnership;
123: 
124:         uint8 numFoundersAdded = 0;
125: 
126:         unchecked {
127:             // For each founder:
128:             for (uint256 i; i < _founders.length; ++i) {
129:                 // Cache the percent ownership
130:                 uint256 founderPct = _founders[i].ownershipPct;
131: 
132:                 // Continue if no ownership is specified
133:                 if (founderPct == 0) {
134:                     continue;
135:                 }
136: 
137:                 // Update the total ownership and ensure it's valid
138:                 totalOwnership += founderPct;
139: 
140:                 // Check that founders own less than 100% of tokens
141:                 if (totalOwnership > 99) {
142:                     revert INVALID_FOUNDER_OWNERSHIP();
143:                 }
144: 
145:                 // Compute the founder's id
146:                 uint256 founderId = numFoundersAdded++;
147: 
148:                 // Get the pointer to store the founder
149:                 Founder storage newFounder = founder[founderId];
150: 
151:                 // Store the founder's vesting details
152:                 newFounder.wallet = _founders[i].wallet;
153:                 newFounder.vestExpiry = uint32(_founders[i].vestExpiry);
154:                 // Total ownership cannot be above 100 so this fits safely in uint8
155:                 newFounder.ownershipPct = uint8(founderPct);
156: 
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
176:             }
177: 
178:             // Store the founders' details
179:             settings.totalOwnership = uint8(totalOwnership);
180:             settings.numFounders = numFoundersAdded;
181:         }
182:     }
```
It is to ensure that tokens are first minted to `_founders[0]` and then minted to other `_founders` in turn.
Therefore, If owner call `updateFounders` function after changing `reservedUntilTokenId` through `Token.sol#setReservedUntilTokenId` function, some of `tokenRecipient` will not be deleted.


## Vulnerability Detail
For instance:
1. Owner initialize `tokenRecipient` through `Token.sol#_addFounders` function using `reservedUntilTokenId`.
2. Owner calls `Token.sol#setReservedUntilTokenId` function and update `reservedUntilTokenId` state variable.
3. Owner calls `Token.sol#updateFounders` function and this function try to delete `tokenRecipient` using the same algorithm with `_addFounders` function. But since `reservedUntilTokenId` has been modified already, some of `tokenRecipient` variables will not be deleted.
5. When `Token.sol#_mintWithVesting` is called to mint new token, by `_isForFounder` function, tokens are minted to the founders who have been removed.


## Impact
If owner calls `updateFounders` after `reservedUntilTokenId` has been changed, tokens are minted to the founders who have been removed.


## Code Snippet
https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L500


## Tool used
Manual Review


## Recommendation
`Token.setReservedUntilTokenId()` should be modified as follows.
```solidity
File: Token.sol
486:     function setReservedUntilTokenId(uint256 newReservedUntilTokenId) external onlyOwner {

+++         if (settings.numFounders > 0) {
+++             revert CANNOT_CHANGE_RESERVE();
+++         }

487:         // Cannot change the reserve after any non reserved tokens have been minted
488:         // Added to prevent making any tokens inaccessible
489:         if (settings.mintCount > 0) {
490:             revert CANNOT_CHANGE_RESERVE();
491:         }
492: 
493:         // Cannot decrease the reserve if any tokens have been minted
494:         // Added to prevent collisions with tokens being auctioned / vested
495:         if (settings.totalSupply > 0 && reservedUntilTokenId > newReservedUntilTokenId) {
496:             revert CANNOT_DECREASE_RESERVE();
497:         }
498: 
499:         // Set the new reserve
500:         reservedUntilTokenId = newReservedUntilTokenId;
501: 
502:         emit ReservedUntilTokenIDUpdated(newReservedUntilTokenId);
503:     }
```