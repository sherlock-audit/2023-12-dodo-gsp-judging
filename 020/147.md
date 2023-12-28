Shambolic Gauze Koala

medium

# Unclear Reentrency implementation

## Summary
In the contract `GSPFunding.sol` , The` nonReentrant` keyword is used Without any proper imports from openzepplin  or modifier, 

## Vulnerability Detail

In the `GSPFunding` contract the` buyShares function` as well as` sellShares function` have`  nonReentrant` modifier but neither the openzepplin reentrency guard is not imported nor a nonReentrant modifier is written in the contract.
## Impact
Loss/Sealing  of funds due to Reentrecny attacks 
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L33
  ```solidity
  function buyShares(address to)
        external
        nonReentrant
        returns (
            uint256 shares,
            uint256 baseInput,
            uint256 quoteInput
        )
    {

```
also here
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L99
## Tool used

Manual Review

## Recommendation
Use openzepplin ReentrancyGuard to protect from reentrency attacks .