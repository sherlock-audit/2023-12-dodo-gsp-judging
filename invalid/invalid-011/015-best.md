Polite Gunmetal Hawk

medium

# No slippage in function `sellBase` and `sellQuote`

## Summary

There is a potential issue in functions `sellBase` and `sellQuote` as they lack slippage checks, which could lead to users experiencing token loss.

## Vulnerability Detail

When a user sells Base or Quote tokens, the contract does not guarantee that the user will receive enough tokens. The absence of slippage checks in `sellBase` and `sellQuote` can result in unexpected token amounts.

Consider the following scenario:
- `USER` holds 2000 shares and there are 2000 BSEE left
- `USER` observes that `OTHER` wants to sell QUOTE, then `USER` sell his shares, now `USER` holds 1 share, and there is only 1 wei BASE left.
- When `OTHER` sells QUOTE, regardless of the quantity of QUOTE sold, they can only receive a maximum of 1 wei BASE.

Add the testcase to `dodo-gassaving-pool/test/GSPFunding.t.sol` to simulate this scenario.

```diff
diff --git a/dodo-gassaving-pool/test/GSPFunding.t.sol b/dodo-gassaving-pool/test/GSPFunding.t.sol
index 10ddcf9..3fa5213 100644
--- a/dodo-gassaving-pool/test/GSPFunding.t.sol
+++ b/dodo-gassaving-pool/test/GSPFunding.t.sol
@@ -35,9 +35,30 @@ contract TestGSPFunding is Test {
         // transfer some tokens to USER
         vm.startPrank(DAI_WHALE);
         dai.transfer(USER, BASE_RESERVE + BASE_INPUT);
+        dai.transfer(OTHER, BASE_RESERVE + BASE_INPUT);
         vm.stopPrank();
         vm.startPrank(USDC_WHALE);
-        usdc.transfer(USER, QUOTE_RESERVE + QUOTE_INPUT);
+        usdc.transfer(USER, 1e10);
+        usdc.transfer(OTHER, 1e10);
+        vm.stopPrank();
+    }
+
+
+    function test_sellQuoteWithoutSlippage() public {
+        vm.startPrank(USER);
+        // init
+        dai.transfer(address(gsp), 2000);
+        usdc.transfer(address(gsp), 2e9);
+        gsp.buyShares(USER);
+        // left only one base token
+        gsp.sellShares(1999, USER, 0, 0, "", block.timestamp);
+        vm.stopPrank();
+
+        vm.startPrank(OTHER);
+        usdc.transfer(address(gsp), 1e7);
+        gsp.sellQuote(OTHER);
+        console.log("user base balance ", dai.balanceOf(OTHER));
+        console.log("user quote balance ", usdc.balanceOf(OTHER));
         vm.stopPrank();
     }

```

Run it wiht `forge test --match-test test_sellQuoteWithoutSlippage  --fork-url $ETH_RPC_URL -vvv`, which reveals that the `OTHER` user gets 1 wei DAI and loses 10 USDC.

```solidity
[PASS] test_sellQuoteWithoutSlippage() (gas: 273570)
Logs:
  user base balance  11000000000000000001
  user quote balance  9990000000
```

## Impact

Users may receive fewer tokens than expected when selling Quote or Base tokens.

## Code Snippet

- [`sellBase`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L40)
- [`sellQuote`](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L79)

## Tool used

Manual Review, Foundry

## Recommendation

It is recommended to add slippage checks to functions `sellQuote` and `sellBase`.