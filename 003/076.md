Small Olive Bird

medium

# Missing Lower Bound Check for Parameter k in init Function

## Summary

see vulnerability details 

## Vulnerability Detail

in the function `init` exactlly this part :

```solidity
// k should be greater than 0 and less than 1018
require(k <= 1018);
K = k;
```
here the  k should be both greater than 0 and less than 10^18, and the require statement only checks that k is less than or equal to 10^18, and does not verify that k is greater than 0, the problem is  k being zero would negatively affect the contract execution.  
this is the expected code 
`require(k > 0 && k <= 10**18, “Invalid K value: must be greater than 0 and less than or equal to 1e18”)`

## Impact

- itialize the GSP with a k value of zero, potentially leading to divisions by zero or other aberrant behaviors in subsequent calculations or financial logic within the contract.

## Code Snippet

- https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol#L34-L97
- https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol#L56-L60
## Tool used
Manual Review
## Recommendation
- it's need a check to ensure that k is greater than zero.
as  `require(k > 0 && k <= 10**18, “Invalid K value: must be greater than 0 and less than or equal to 1e18”)`