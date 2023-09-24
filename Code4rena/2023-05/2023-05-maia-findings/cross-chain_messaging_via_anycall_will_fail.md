## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-34

# [Cross-chain messaging via Anycall will fail](https://github.com/code-423n4/2023-05-maia-findings/issues/91) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/BranchBridgeAgent.sol#L1006-L1011
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/lib/AnycallFlags.sol#L11


# Vulnerability details

## Impact
Cross-chain calls will fail since source-fee is not supplied to Anycall

## Proof of Concept
In `_performCall()` of BranchBridgeAgent.sol, a cross-chain called is made using `anyCall()` with the `_flag` of [4](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/lib/AnycallFlags.sol#L11).  According to the [Anycall V7 documentation](https://docs.multichain.org/developer-guide/anycall-v7/how-to-integrate-anycall-v7#request-parameters) and [code](https://github.com/anyswap/multichain-smart-contracts/blob/645d0053d22ed63005b9414b5610879094932304/contracts/anycall/v7/AnycallV7Upgradeable.sol#L205-L207), when using gas `_flag` of 4, the gas fee must be paid on the source chain. This means `anyCall()` must be called and sent gas. 

However, this is not the case, and the result is `_performCall` will always revert. This will impact many functions that rely on this function such as `callOut()`, `callOutSigned()`,  `retryDeposit()`, and etc.

## Tools Used
Manual

## Recommended Mitigation Steps
After discussing with the Sponsor, it is expected that the fee be paid on the destination chain, specifically the `rootBridgeAgent`. Consider refactoring the code to change the `_flag` to use [pay on destination](https://github.com/anyswap/multichain-smart-contracts/blob/645d0053d22ed63005b9414b5610879094932304/contracts/anycall/v7/interfaces/AnycallFlags.sol#L9).

Alternatively, if pay on source is the intention, consider refactoring the code to include fees, starting with `_performCall`. Additional refactoring will be required.
```
function _performCall(bytes memory _calldata, uint256 _fee) internal virtual {
    //Sends message to AnycallProxy
    IAnycallProxy(localAnyCallAddress).anyCall{value: _fee}(
        rootBridgeAgentAddress, _calldata, rootChainId, AnycallFlags.FLAG_ALLOW_FALLBACK, ""
    );
}
```









## Assessed type

Library