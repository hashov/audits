# First Flight #12: Kitty Connect - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Missed adding "tokenId" to the "s_ownerToCatsTokenId" when "mintBridgedNFT()"](#H-01)
    - ### [H-02. "kittyTokenCounter" is only increasing and never decreasing](#H-02)
    - ### [H-03. Owner approval is never made making "safeTransferFrom()" unusable](#H-03)
- ## Medium Risk Findings
    - ### [M-01. "Idx" of burned nft won't be switched with "lastItem" popped out of "s_ownerToCatsTokenId"](#M-01)
- ## Low Risk Findings
    - ### [L-01. "mintCatToNewOwner()" with "dob" parameter, might lead to overflow/underflow](#L-01)
    - ### [L-02. Constructor missing zero address check for "i_kittyConnectOwner"](#L-02)
    - ### [L-03. Unused errors inside "KittyBridgeBase.sol"](#L-03)
    - ### [L-04. Storage variables "s_linkToken", "kittyConnect", "gasLimit" can be made immutable](#L-04)
    - ### [L-05. Magic numbers usage for "gasLimit" storage variable inside the constructor of KittyBridge.sol](#L-05)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #12: Kitty Connect

### Dates: Mar 28th, 2024 - Apr 4th, 2024

[See more contest details here](https://www.codehawks.com/contests/clu7ddcsa000fcc387vjv6rpt)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 1
   - Low: 5


# High Risk Findings

## <a id='H-01'></a>H-01. Missed adding "tokenId" to the "s_ownerToCatsTokenId" when "mintBridgedNFT()"            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L154

## Summary
Missed adding "tokenId" to the "s_ownerToCatsTokenId" when "mintBridgedNFT()".

## Vulnerability Details
When "mintBridgedNFT()" function is called it is forgotten to add the "tokenId" to the list that keeps track of the nfts of the owner 
 "s_ownerToCatsTokenId" resulting to inconsistencies when bridging.

## Impact
High, since this token won't be tracked as part of the entries of the owner, resulting to inconsistencies when bridging.

## Tools Used
Manual review

## Recommendations
Line such as this to be added just before emitting the "NFTBridged" event:

```solidity
s_ownerToCatsTokenId[catOwner].push(tokenId);
```
## <a id='H-02'></a>H-02. "kittyTokenCounter" is only increasing and never decreasing            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L165

https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L96

https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L146

## Summary
"kittyTokenCounter" is only increasing and never decreasing. 

## Vulnerability Details
"kittyTokenCounter" is only increasing and never decreasing even when we are burning/transfering tokens from this contract to another, meaning we will have some owners with s_ownerToCatsTokenId = (10,20,13,6,3) meaning this array is not guaranteed to be ordinal and in a sequence. Therefore this code piece will break the business logic:

```solidity
        if (idx < (userTokenIds.length - 1)) {
            s_ownerToCatsTokenId[msg.sender][idx] = lastItem;
        }
```
As previously submitted this finding with "s_ownerToCatsTokenId"   being in sequence it is possible for them not to be in a sequence therefore idx to be huge amount than the "s_ownerToCatsTokenId.length"  meaning it will never go in the if, therefore never removing the "idx" that we are burning and again losing an nft for the owner.
 
## Impact
High, since the owner of the nft loses one of his nfts and remains with a reference to the id of the one that is burned.

## Tools Used
Manual review.

## Recommendations
Remove the if and do the following:
```solidity
if(lastItem != idx){
    s_ownerToCatsTokenId[msg.sender].pop()
    ownerToCatsTokenId[msg.sender][idx] = lastItem;
} else {
        s_ownerToCatsTokenId[msg.sender].pop()
}
```

## <a id='H-03'></a>H-03. Owner approval is never made making "safeTransferFrom()" unusable            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L121

## Summary
Owner approval is never made making "safeTransferFrom()" unusable inside of "KittyConnect.sol".

## Vulnerability Details
Owner approval is never made making "safeTransferFrom()" unusable inside of "KittyConnect.sol". When entering the "safeTransferFrom()" method the business logic leads to a check such as:
```solidity
require(getApproved(tokenId) == newOwner, "KittyConnect__NewOwnerNotAppr
```
This code will always throw an error and will revert the transaction since no publicly exposed function for granting permission to the new owner from the old owner is present, therefore cannot be called explicitly before "safeTransferFrom()" so the above mentioned require check to pass, OR no call to function "approve(address to, uint256 tokenId)" from ERC721 contract is made implicitly inside of "safeTransferFrom()" (even though that will also break the invariant, because it is stated that the owner of the cat has to give the approval to the new owner, and not the shop partner, but still that will make the contract transfers possible) right before the above mentioned check resulting in granting access to the new owner. This vulnerability breaks the following invariant of the protocol: "* @notice but requires the approval of the cat owner to the new owner before shop partner calls this" which is stated in the doc above the "safeTransferFrom()";

## Impact
High, since the safeTranferFrom() function inside the protocol will always revert, making the transfers impossible and protocol unusable.

## Tools Used
Manual review.

## Recommendations
Expose an external function such as:
```solidity
    function grantPermissions(address newOwner, uint256 tokenId) external {
        require(_ownerOf(tokenId) == msg.sender, "KittyConnect__NotKittyOwner");
        approve(newOwner, tokenId);

        emit ApprovFromOwnerGranted(msg.sender, newOwner, tokenId);
    }
```

		
# Medium Risk Findings

## <a id='M-01'></a>M-01. "Idx" of burned nft won't be switched with "lastItem" popped out of "s_ownerToCatsTokenId"            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L146

## Summary
"Idx" of burned nft won't be switched with "lastItem" popped out of "s_ownerToCatsTokenId" if idx itself is the last item inside of the array after the previous last item was popped out.

## Vulnerability Details
Inside "KittyConnect.sol" inside "bridgeNftToAnotherChain()" the "Idx" of burned nft won't be switched with "lastItem" popped out of "s_ownerToCatsTokenId" if idx itself is the last item inside of the array after the previous last item was popped out. To make it more clear if we have s_ownerToCatsTokenIds filled with ids as such (1,2,3,4) and the idx of the nft owner wants to transfer to be equal to 3 (idx=3 with index=2, cause arrays in Solidity are 0 indexed) and we follow the business logic:
```solidity
        uint256[] memory userTokenIds = s_ownerToCatsTokenId[msg.sender];
        uint256 lastItem = userTokenIds[userTokenIds.length - 1]; //Last item represents last token Id of the owner, so lastItem = userTokenIds.length(which is //of value 4 - 1, meaning lastItem will be at index 3 meaning value=4;

        s_ownerToCatsTokenId[msg.sender].pop(); //this removes the last element of the array with "value=4 and index 3" of token Ids of the owner, and //leaves the array with total of 3 elements (1,2,3)

     if (idx < (userTokenIds.length - 1)) { //This is where vulnerability occurs
            s_ownerToCatsTokenId[msg.sender][idx] = lastItem;
        }
```
to continue the example off-code our idx=3 which is not smaller than 2 (userTokenIds.length is 3-1) meaning we will not go in the if and value=4 (lastItem) will not get to the place of the idx, which leads to the idx staying in the "s_ownerToCatsTokenId" and lastItem being lost.

## Impact
High, since the owner of the nft loses one of his nfts and remains with a reference to the id of the one that is burned.

## Tools Used
Manual review.

## Recommendations 
The equation should use equal sign as well in the if statement so:
```solidity
if (idx <= (userTokenIds.length - 1)) 
```

# Low Risk Findings

## <a id='L-01'></a>L-01. "mintCatToNewOwner()" with "dob" parameter, might lead to overflow/underflow            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L92

## Summary
"mintCatToNewOwner()" with "dob" parameter, might lead to overflow/underflow.

## Vulnerability Details
"mintCatToNewOwner()" with "dob" parameter, might lead to overflow/underflow, inside getCatAge() since there is this piece of code there:

```solidity
block.timestamp - s_catInfo[tokenId].dob;
```
resulting a revert inside getCatAge(), depending on what has been passe initially inside "mintCatToNewOwner()";

## Impact
Medium, nobody will be able to fetch the given cat age.

## Tools Used
Manual review.

## Recommendations
Do not pass "dob" as parameter to mintCatToNewOwner() instead keep it internal, cause obviously the dateOfBirth can be the time when mintCatToNewOwner() is called so "dob: block.timestamp" inside the "CatInfo" struct will do the job.
## <a id='L-02'></a>L-02. Constructor missing zero address check for "i_kittyConnectOwner"            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L67

## Summary
Constructor missing zero address check for "i_kittyConnectOwner" inside of "KittyConnect.sol".

## Vulnerability Details
The missing zero address check is present, which means can instantiate the contract without specifying address and will lose ownership of the contract.

## Impact
High, since owner will lose ownership of the contract, but likelihood of that happening is minimum, cause everyone watches what he deploys usually.

## Tools Used
Manual review.

## Recommendations
Have a code piece inside of the contract initialization similar to:
```solidity
 require(msg.sender != address(0), "Address 0 not allowed!");
```

## <a id='L-03'></a>L-03. Unused errors inside "KittyBridgeBase.sol"            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/base/KittyBridgeBase.sol#L10

https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/base/KittyBridgeBase.sol#L11

## Summary
Errors "KittyBridge__NothingToWithdraw" and "KittyBridge__FailedToWithdrawEth" inside of "KittyBridgeBase.sol" are not being used anywhere.
## Vulnerability Details
Errors "KittyBridge__NothingToWithdraw" and "KittyBridge__FailedToWithdrawEth" inside of "KittyBridgeBase.sol" are not being used anywhere, have to be removed for gas cost optimization.

## Impact
Low.

## Tools Used
Manual review.

## Recommendations
Remove the errors for gas cost optimization.

## <a id='L-04'></a>L-04. Storage variables "s_linkToken", "kittyConnect", "gasLimit" can be made immutable            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/base/KittyBridgeBase.sol#L33

https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/base/KittyBridgeBase.sol#L34

## Summary
Storage variables "s_linkToken", "kittyConnect", "gasLimit" can be made immutable.

## Vulnerability Details
Those variables are getting initialized once in the constructor on the contract initialization therefore having them as immutable will save the contract some gas and will apply best practices.

## Impact 
Low

## Tools Used
Part Slither and part manual review.

## Recommendations
Make storage variables immutable and add "i_" before the name to scope with best practices for naming conventions.
## <a id='L-05'></a>L-05. Magic numbers usage for "gasLimit" storage variable inside the constructor of KittyBridge.sol            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyBridge.sol#L29

## Summary
Magic numbers for "gasLimit" storage variable inside the constructor of "KittyBridge.sol"

## Vulnerability Details
As we know magic numbers are hardly against best practices, so usage of constants is highly encouraged, this way we will have 
a constant as one source of a truth for every usage of the variable and also will save gas for the protocol since constants in Solidity are typed in the bytecode directly and do not get stored in the storage.

## Impact
Low, since it's gas related issue and would be a big impact for example, if an extra zero is being added to the variable.

## Tools Used
Slither

## Recommendations
Use constant such as:
uint256 internal constant GAS_LIMIT = 400000;



