## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-24

# [Malicious user can set any contract as local hToken for an underlying since there is no access control for "_addLocalToken"](https://github.com/code-423n4/2023-05-maia-findings/issues/285) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/CoreBranchRouter.sol#L78


# Vulnerability details

## Impact
Malicious user can deliberately set a irrelevant (or even poisonous) local hToken for an underlying token, as anyone can directly access `_addLocalToken` at root chain without calling `addLocalToken` at branch chain first.

## Proof of Concept
    function addLocalToken(address _underlyingAddress) external payable virtual {
        //Get Token Info
        string memory name = ERC20(_underlyingAddress).name();
        string memory symbol = ERC20(_underlyingAddress).symbol();

        //Create Token
        ERC20hToken newToken = ITokenFactory(hTokenFactoryAddress).createToken(name, symbol);

        //Encode Data
        bytes memory data = abi.encode(_underlyingAddress, newToken, name, symbol);

        //Pack FuncId
        bytes memory packedData = abi.encodePacked(bytes1(0x02), data);

        //Send Cross-Chain request (System Response/Request)
        IBridgeAgent(localBridgeAgentAddress).performCallOut{value: msg.value}(msg.sender, packedData, 0, 0);
    }

The intended method to add a new local token for an underlying is by calling function `addLocalToken` at the branch chain. However, it appears that the last line of code, `IBridgeAgent(localBridgeAgentAddress).performCallOut{value: msg.value}(msg.sender, packedData, 0, 0);` uses `performCallOut` instead of `performSystemCallOut`. This means that users can directly `callOut` at branch bridge agent with `_params = abi.encodePacked(bytes1(0x02), abi.encode(_underlyingAddress, anyContract, name, symbol))` to invoke `_addLocalToken` at the root chain without calling `addLocalToken` first. As a result, they may set an arbitrary contract as the local token. It's worth noting that the impact is irreversible as there is no mechanism to modify or delete local tokens, meaning that the underlying token can never be properly bridged in the future.

The branch hToken is called by function `bridgeIn` when `redeemDeposit` or `clearToken`:

    function bridgeIn(address _recipient, address _localAddress, uint256 _amount)
        external
        virtual
        requiresBridgeAgent
    {
        ERC20hTokenBranch(_localAddress).mint(_recipient, _amount);
    }
Below are several potetial exploiting methods:
1. If a regular ERC20 contract with admin minting permissions is set, exploiter can mint unlimited amount of local token for himself. By bridging them, he can receive an arbitrary amount of global token at the root chain.
2. If an unrelated contract with empty `mint` function is set, the underlying asset would be unable to be bridged in from the root chain, and users who attempt to do so could lose their assets.
3. If a malicious contract is set, gas griefing is possible.
4. This contract may serve as an intermediary for reentrancy. (I haven't found a concrete way so far but there is a potential risk)

## Tools Used
Manual

## Recommended Mitigation Steps
Use `performSystemCallOut` and `executeSystemRequest` to send Cross-Chain request for adding local token.








## Assessed type

Access Control