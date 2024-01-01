Clumsy Dijon Halibut

medium

# `InitializableOwnable` ownership can be changed immediately by malicious owner

## Summary

The `InitializableOwnable` contract has vulnerabilities related to ownership management. Firstly, it allows for immediate ownership changes by the owner without adhering to the intended timelock delay. Secondly, it employs an `onlyModifier` access control modifier, which presents centralization risks, which I've decided to dedicate a separate [issue](https://github.com/sherlock-audit/2023-12-dodo-gsp-rickardlarsson22/issues/3).

## Vulnerability Detail

### Immediate Ownership Change

The `InitializableOwnable` contract has a function called [transferOwnership()](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/InitializableOwnable.sol#L47-L50) which permits the current owner to transfer ownership without any delay or additional confirmation. 

[https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/InitializableOwnable.sol#L47-L50](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/InitializableOwnable.sol#L47-L50)
```solidity
File: dodo-gassaving-pool/contracts/lib/InitializableOwnable.sol

47:    function transferOwnership(address newOwner) public onlyOwner {
48:        emit OwnershipTransferPrepared(_OWNER_, newOwner);
49:        _NEW_OWNER_ = newOwner;
50:    }
```

This vulnerability exposes the contract to potential security risks, as the owner or a malicious actor with access to the owner's account can take control of the contract immediately, bypassing all intended security measures.

## Impact 

The vulnerability allows unauthorized ownership changes, potentially enabling a malicious actor to quickly take control of the contract with little opportunity for intervention or verification.

## Code Snippet

[https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/InitializableOwnable.sol#L47-L50](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/InitializableOwnable.sol#L47-L50)

[https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/InitializableOwnable.sol#L55](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/lib/InitializableOwnable.sol#L55)

## Tool used

Manual Review

## Recommendation

Implement a two-step process for transferring ownership. First, prepare ownership transfer and signal intent. Second, wait for specified timelock delay before claiming ownership.