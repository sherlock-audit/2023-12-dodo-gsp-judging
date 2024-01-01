Oblong Holographic Mallard

high

# User can lose fund when trying to call sellShares()

## Summary

Function Declaration: sellShares(
uint256 shareAmount, address to ,uint256 baseMinAmount ,uint256 quoteMinAmount ,bytes calldata data ,uint256 deadline
) 

In GSPFunding.sol, the function sellShares() allows users to specify baseMinAmount and quoteMinAmount. However, there's no subsequent check to verify if these values are greater than zero.

Validation in line 120:  require(
baseAmount >= baseMinAmount && quoteAmount >= quoteMinAmount,"WITHDRAW_NOT_ENOUGH"
);

The validation in line 120 ensures that baseAmount and quoteAmount are at least as much as baseMinAmount and quoteMinAmount respectively. Yet, this doesn’t guarantee these minimum values aren’t set to zero.

As this function's purpose is to burn the user's shares and grant them the baseToken and quoteToken, having a minimum amount of 0 allows potential exploitation. Specifically, a user might get exploited by a malicious actor, resulting in receiving zero tokens despite holding a considerable shareAmount. This scenario exposes the user to front-running or being sandwiched, leading to a loss even if they possess a valuable shareAmount.



## Vulnerability Detail

Proof of Concept:

1. ALICE initiates gsp.sellShares(shareAmount, ALICE, 0, 0, "", block.timestamp + 5).
2. The transaction enters the mempool and the attacker observes it .
3. The attacker promptly front-runs the transaction by executing a substantial DAI/USDC swap using buyShares().
4. ALICE sells shares at an elevated price.
5. ALICE receives 0 tokens, contrary to the expected amount, as she specified baseMinAmount and quoteMinAmount as 0.
6. Subsequently, the attacker executes the reverse transaction, maximizing profit by sandwiching ALICE's transaction.

## Impact

The risk is high as user funds might be compromised due to incorrect parameter input.

## Code Snippet
    
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L92-L146

https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/main/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L120-L123


## Tool used

Manual Review

## Recommendation
Ensure correct validation for baseMinAmount and quoteMinAmount, confirming they exceed zero.