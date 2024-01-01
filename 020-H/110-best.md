Witty Cerulean Panther

high

# Liquidity incentive for depositing liquidity is wrong

## Summary
When users deposits tokens to mint shares the incentive for providing the scarce token is not existing. Contrary, the incentive is given to the most amounted token which incentives users to deposit the token that has more in the pool. 
## Vulnerability Detail
Let's consider an example where a pool initially has 100-100 tokens at time t=0, with a total of 100 shares in totalSupply, representing perfect equilibrium.

After several trades, the token balances in the pool change to 20-180, indicating that the base token is scarce while the quote token is in abundance within the pool. This situation should ideally incentivize users to deposit base tokens in order to receive more LP tokens since the pool is not in equilibrium. However, this doesn't occur due to the way the share calculation operates when a user deposits tokens into the pool:
```solidity
uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio;
// The shares will be minted to user
shares = DecimalMath.mulFloor(totalSupply, mintRatio);
```
As observed, the lowest ratio is chosen. Considering we had 20-180 tokens in the pool, if a user intends to deposit a total of 100 tokens, what would be the optimal deposit strategy? Depositing 90 base and 10 quote tokens would increase the base tokens and move the pool towards equilibrium. Let's see how many shares the user would receive by depositing 90-10:

mintRatio = min(90/20, 10/180)
= 10/180

10/180 * 100 = 5.55 shares.

Now, what if the user deposits liquidity in reverse, with 10 base tokens and 90 quote tokens?

mintRatio = min(10/20, 90/180)
= 1/2

1/2 * 100 = 50 shares!

Thus, depositing 10-90, despite the pool having significantly more quote tokens, would grant the user nearly 10 times more shares. Users incentivized to provide liquidity deviate the pool further from its balanced state, which contradicts the pool's equilibrium!

Here a coded PoC:
```solidity
function test_DepositInFavourOfThePool() external {
        // tapir deposits 100-100 to initiate the pool
        // assume the pool has 20-180, dont want to do sells to further complexify the issue
        // so I'll start the pool with 20-180 from start. 
        vm.startPrank(tapir);
        dai.safeTransfer(address(gsp), 20 * 1e18);
        usdc.transfer(address(gsp), 180 * 1e6);
        gsp.buyShares(tapir);
        
        // deposit 90 base and 10 quote, which is better for the pools equilibrum
        dai.safeTransfer(address(gsp), 90 * 1e18);
        usdc.transfer(address(gsp), 10 * 1e6);
        (uint256 shares, , ) = gsp.buyShares(tapir);

        console.log("Tapir received shares", shares);
        console.log("Tapir received shares human readable form", shares / 1e18);
    }

    function test_DepositNotInFavourOfThePool() external {
        // tapir deposits 100-100 to initiate the pool
        // assume the pool has 20-180, dont want to do sells to further complexify the issue
        // so I'll start the pool with 20-180 from start. 
        vm.startPrank(tapir);
        dai.safeTransfer(address(gsp), 20 * 1e18);
        usdc.transfer(address(gsp), 180 * 1e6);
        gsp.buyShares(tapir);
        
        // deposit 10 base and 90 quote, which is worse for the pools equilibrum
        dai.safeTransfer(address(gsp), 10 * 1e18);
        usdc.transfer(address(gsp), 90 * 1e6);
        (uint256 shares, , ) = gsp.buyShares(tapir);

        console.log("Tapir received shares", shares);
        console.log("Tapir received shares human readable form", shares / 1e18);
    }
```

Test result and logs:
<img width="501" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/334dba64-2a80-4a41-9344-80670aa8f438">

## Impact
High, since the pool is never incentivized to be in equilibrium when users provide liquidity. Depositing the scarce token should give the user a bonus on shares since the user helps the pool to reach its equilibrium.
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L31-L82
## Tool used

Manual Review

## Recommendation
Change the mintRatio calculation