Witty Cerulean Panther

high

# RState can be inconsistent with the mathematical formula because of the LP_FEES

## Summary
The "R" parameter plays a pivotal role in swaps as it governs the rules dictating how the swaps should execute. However, in certain cases, the R value might not align with the documentation due to LP fees deducted from users. When R is incorrectly set, the swaps become inaccurate and follow an unintended formula, leading to discrepancies.
## Vulnerability Detail
The R value can take three different values: R < 1, R = 1, and R > 1. Each value has its own set of rules that determine the value of R.
The DODO documentation outlines the determination of the R value:
<img width="623" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/3647a6cd-26e7-4813-88bc-893ab6056d3a">

When the base reserve, base target, quote reserve, and quote target are equal to each other, R = 1. However, the math in DODOMath does not include the LP fee, which impacts the determination of base/quote reserves.
During a quote, the actual amount received by the trader is calculated using DODOMath, after which mtFeeRate and lpFeeRate are deducted from the trader's receivable amount, as seen in the following snippet:
```solidity
mtFee = DecimalMath.mulFloor(receiveQuoteAmount, mtFeeRate);
receiveQuoteAmount = receiveQuoteAmount
- DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate)
- mtFee;
 newBaseTarget = state.B0;
```

Throughout the codebase, the MT_FEE is never accounted for in the reserves and is always managed separately by subtracting the MT_FEE from the total contract balance. Conversely, LP fees are accounted for and added to the reserves. When a trader sells quote/base tokens, they pay the MT_FEE to the owner and the LP_FEE to the LP'ers, which are reflected in the reserves. This differentiation can be observed in the following snippet:
```solidity
_setReserve((_BASE_TOKEN_.balanceOf(address(this)) - _MT_FEE_BASE_), quoteBalance);
// from the sellQuote function, but same applies for sellBase too. The LP fee is added to the reserves but MT_FEE is not.
```

For instance, in a DAI-USDC pool starting with 10 DAI and 10 USDC, selling 5 base tokens (DAI) to the pool in exchange for quote tokens leads to R < 1. To revert R back to 1, an exact amount of quote tokens is required to make Q0 equal to Q (Q0 - Q). Achieving this equalization between reserves and their targets restores R to 1.

However, it's important to note that all tests conducted in the codebase assume an LP_FEE of 0, which is unrealistic as it implies no earnings for the LP providers. When the LP_FEE is a different value, inconsistencies arise. If there were an LP fee, the reserves and their targets can be different and not inline with the mathematical definition of R! 

Now, considering these factors, let's take an example where R will be calculated as ONE, even though B0 != B and Q0 == Q.
A PoC demonstrates that the reserves and their targets are not equal, yet R = 1:
```solidity
            uint256 backToOnePayQuote = state.Q0 - state.Q;
            uint256 backToOneReceiveBase = state.B - state.B0;
            if (payQuoteAmount < backToOnePayQuote) {
                receiveBaseAmount = _RBelowSellQuoteToken(state, payQuoteAmount);
                newR = RState.BELOW_ONE;
                if (receiveBaseAmount > backToOneReceiveBase) {
                    receiveBaseAmount = backToOneReceiveBase;
                }
            } else if (payQuoteAmount == backToOnePayQuote) {
                receiveBaseAmount = backToOneReceiveBase;
                newR = RState.ONE;
            } else {
                receiveBaseAmount = backToOneReceiveBase + (
                    _ROneSellQuoteToken(state, payQuoteAmount - backToOnePayQuote)
                );
                newR = RState.ABOVE_ONE;
            }
        }
```


Here a PoC demonstrates that the reserves and their targets are not equal but the R = 1 
```solidity
function test_RState_Different() public {
        vm.startPrank(tapir);

        //  Buy shares with tapir, 10 - 10
        dai.safeTransfer(address(gsp), 10 * 1e18);
        usdc.transfer(address(gsp), 10 * 1e6);
        gsp.buyShares(tapir);

        console.log("MT_FEE_QUOTE", gsp._MT_FEE_QUOTE_());
        console.log("MT_FEE_BASE", gsp._MT_FEE_BASE_());
        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());
        console.log("RState", gsp._RState_());

        assertTrue(gsp._BASE_RESERVE_() == 10 * 1e18);
        assertTrue(gsp._QUOTE_RESERVE_() == 10 * 1e6);
        assertTrue(gsp._BASE_TARGET_() == 10 * 1e18);
        assertTrue(gsp._QUOTE_TARGET_() == 10 * 1e6);
        assertEq(gsp.balanceOf(tapir), 10 * 1e18);
        vm.stopPrank();
        
        // first sell some base tokens to move the price
        vm.startPrank(hippo);
        dai.transfer(address(gsp), 5 * 1e18);
        uint256 receivedQuoteAmount = gsp.sellBase(hippo);

        console.log("Received quote amount by hippo", receivedQuoteAmount);
        console.log("MT_FEE_QUOTE", gsp._MT_FEE_QUOTE_());
        console.log("MT_FEE_BASE", gsp._MT_FEE_BASE_());
        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());
        console.log("USDC balanceOf", usdc.balanceOf(address(gsp)));
        console.log("DAI balanceOf", dai.balanceOf(address(gsp)));
        console.log("RState", gsp._RState_());

        // @review deploy the GSP with this value!
        //uint256 LP_FEE_RATE = 100000000000000; = 0.0001%
        uint256 backTo1Quote = gsp._QUOTE_TARGET_() - gsp._QUOTE_RESERVE_();
        backTo1Quote += DecimalMath.mulFloor(backTo1Quote, LP_FEE_RATE);
        
        // sell some amount of quote such that the inconsistency can be observed, this is usually the lp fee +
        usdc.transfer(address(gsp), backTo1Quote);
        uint256 receivedBaseAmount = gsp.sellQuote(hippo);


        // if RState is 0 (means R = 1 in enum value) then check the reserves and their targets we expect them to be equal to each other when r = 1 !!!
        console.log("Received quote amount by hippo", receivedBaseAmount);
        console.log("MT_FEE_QUOTE", gsp._MT_FEE_QUOTE_());
        console.log("MT_FEE_BASE", gsp._MT_FEE_BASE_());
        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());
        console.log("USDC balanceOf", usdc.balanceOf(address(gsp)));
        console.log("DAI balanceOf", dai.balanceOf(address(gsp)));
        console.log("RState", gsp._RState_());
```

## Impact
Since this is a bug in the core logic of swaps I will label this as high. This will make the swaps happening in the pool with a different formula because R is calculated with mistaken reserves.
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L224-L244
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/PMMPricing.sol#L43-L123
## Tool used

Manual Review

## Recommendation
Account the LP_FEES when determining which value of swap will change the "R" value.  