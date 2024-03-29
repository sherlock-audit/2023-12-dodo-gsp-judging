Bitter White Ape

medium

# AdjustedTarget Does not Update When RState is ONE

## Summary

## Vulnerability Detail

The RState is changed at the end of each `sell` Quote or Base token. Then, at the start of the next swap, it is set within `querySellQuote` -> `getPMMState`.

If the `RState` at the end of the swap is `ONE`, then the `adjustedTarget` function does not do anything as noth the `if (state.R == RState.BELOW_ONE)` and `else if (state.R == RState.ABOVE_ONE)` conditions are `false`. This means, the Q0 and B0 `variables` not updated and retain the un-updated state before the last swap that happened.

This results in a different price curve and can be taken advantage of to make there be less/more liquidity than there should during a swap.

## Impact

The liquidity curve does not get updated based on `RState` of `ONE` and results in an incorrect liquidity curve and price impacts for swappers.

## Code Snippet

https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/lib/PMMPricing.sol#L237-L253


## Tool used

Manual Review

## Recommendation

Add a condition `if RState == ONE` in the `adjustedTarget` function and which updates the `Q0` and `B0` variables