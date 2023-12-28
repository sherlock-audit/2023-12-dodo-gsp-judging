Bitter White Ape

high

# `newBaseTarget` is Based off Outdated State

## Summary

`newBaseTarget` is based off the `state` which has not had the `RState` change applied yet, leading to an incorrect price curve.


## Vulnerability Detail

In the `querySellBase` function the `newBaseTarget` is based off the `state` struct stored in `memory`

```solidity
newBaseTarget = state.B0;
```

defined here:

```solidity
PMMPricing.PMMState memory state = getPMMState();
```

The problem is that the new RState is calculated in the call: 

```solidity
(receiveQuoteAmount, newRState) = PMMPricing.sellBaseToken(state, payBaseAmount);
```

but the RState change is not applied to the `state` struct. That means that the `newBaseTarget` is updated incorrectly. The change in RState is only applied after the `querySellBase` call in the `SellBase` function:

```solidity
        if (_RState_ != uint32(newRState)) {    
            require(newBaseTarget <= type(uint112).max, "OVERFLOW");
            _BASE_TARGET_ = uint112(newBaseTarget);
            _RState_ = uint32(newRState);
            emit RChange(newRState);
        }
```

_Note: This also applies to `QuerySellQuote`_

## Impact

`newBaseTarget` is updated incorrectly which leads to an incorrect AMM price curve.

## Code Snippet

https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L243

## Tool used

Manual Review

## Recommendation

Apply the change to `RState` first, and then retrieve the `newBaseTarget` from `adjustedTarget`