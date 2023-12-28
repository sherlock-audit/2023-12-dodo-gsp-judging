Witty Cerulean Panther

medium

# Division before multiplication can result to rounding errors

## Summary
Throughout the code there are some part of the code that does division before multiplication which could end up in loss of precision. 
## Vulnerability Detail
Let's take an example of this part of the code:
```solidity
uint256 part2 = k * (V0) / (V1) * (V0) + (i * (delta)); // kQ0^2/Q1-i*deltaB
```
so the above math is simply:
kQ^^2 / Q1 - (i * deltaB). So what we should be doing is first getting the square of the Q2 which is V0 in the above code and then multiply it by k. However, the code first does the k * V0 and then divides it to V1 and back to multiplication with V1. If the "k" value is significantly lesser than 1e18, then k * V0 / V1 can result on precision error and even in rounding to "0". 
Assume k = 1, V0 = 50 * 1e18 and V1 = 100 * 1e18, then the result would be:

1 * 50 * 1e18 / 100 * 1e18 = 0 in solidity. Which would result that the entire part2 to be calculated mistakenly. 

Also, we can see that the "k" value can be any value between 0 < 1e18 here:
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol#L58-L59

so the above scenario becomes realistic.
## Impact
Although having a "k" value 1 is not very realistic, it still can happen since the k is in range of 0 < 1e18, in such cases this rounding error will make the amounts to be calculated mistakenly. Hence, I'll label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/DODOMath.sol#L83
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/DODOMath.sol#L147
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/DODOMath.sol#L159
## Tool used

Manual Review

## Recommendation
First multiply then divide throughout in the code