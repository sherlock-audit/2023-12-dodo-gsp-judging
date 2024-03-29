Wobbly Tawny Canary

medium

# Incomplete Fee Rate Control

## Summary

The `GSP.sol` contract's initialization function allows for setting various parameters, including `lpFeeRate` and `mtFeeRate`. 
But there are missing limit checks on them, like there are for the `_I_` and `_K_` parameters. Btw the comment below them is incorrect:

```solidity
        // i should be greater than 0 and less than 10**36 // less than or equal to ...
        require(i > 0 && i <= 10**36);
        _I_ = i;
        // k should be greater than 0 and less than 10**18 // less than or equal to ...
        require(k <= 10**18);
        _K_ = k;
```

However, while there's a designated function `adjustMtFeeRate` in the `GSPVault.sol` contract to modify the `mtFeeRate`, there is no corresponding function provided to adjust the `lpFeeRate`.

## Vulnerability Detail

The absence of limits on `lpFeeRate` and `mtFeeRate` within the `GSP.sol` `init` function poses a risk of unbounded adjustments. This lack of constraints could lead to unintended scenarios where fee rates are set to extreme values, potentially disrupting the protocol's stability and fairness. Moreover, the missing adjustment function for lpFeeRate limits the governance control over this fee rate, creating an inconsistency in managing different fee structures.


```solidity
        // _LP_FEE_RATE_ is set when initialization
        _LP_FEE_RATE_ = lpFeeRate; // @audit-info no limits
        // _MT_FEE_RATE_ is set when initialization
        _MT_FEE_RATE_ = mtFeeRate; 
```

## Impact

This lack of constraints could lead to unintended scenarios where fee rates are set to extreme values, potentially disrupting the protocol's stability and fairness.

## Code Snippet

https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol#L61-L64

https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L177-L184

## Tool used

Manual Review

## Recommendation

Implement upper and lower limits or constraints on both `lpFeeRate` and `mtFeeRate` within the GSP initialization function.

Furthermore, introduce a function analogous to `adjustMtFeeRate` that allows adjusting `lpFeeRate`. 