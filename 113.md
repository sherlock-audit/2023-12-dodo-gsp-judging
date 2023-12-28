Witty Cerulean Panther

medium

# EIP712 chainId is hardcoded

## Summary
Per EIP712, calculating the domain separator using a hardcoded chainId could pose problems. The reason is that if the chain undergoes a hard fork and changes its chain id, the domain separator will be inaccurately calculated. To avoid this issue, the domain separator should be dynamically calculated using the chainId opcode each time it is requested.
## Vulnerability Detail
As stated in the summary, hardcoding the chain ID is not a good practice when implementing EIP712. The best practice is to use a dynamically calculated DOMAIN_SEPARATOR, as suggested by OpenZeppelin.

 Here some previous findings in the space regards to this specific issue:
https://solodit.xyz/issues/eip712-domain_separator-stored-as-immutable-mixbytes-none-bebop-markdown
https://solodit.xyz/issues/m-05-replay-attack-in-case-of-hard-fork-code4rena-golom-golom-contest-git
## Impact
Medium, since this issue was previously counted as medium in various protocols and it can be an issue where permit can be used for infinite approvals if the hard fork chain id suits. Also, DODO deployed in various of chains so the thread of this happening is more likely 
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSP.sol#L86-L94
## Tool used

Manual Review

## Recommendation
build the domain separator dynamically with dynamic block.chainId in case of forks of the chain. Smith like this:
```solidity
function _buildDomainSeparator() private view returns (bytes32) {
        return keccak256(abi.encode(TYPE_HASH, _EIP712NameHash(), _EIP712VersionHash(), block.chainid, address(this)));
    }
```