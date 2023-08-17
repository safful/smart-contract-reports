## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-14

# [Cross-chain replay attacks are possible with `deployLPToken`](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/154) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LPTokenFactory.sol#L27-L48


# Vulnerability details

### Impact
Mistakes made on one chain can be re-applied to a new chain

There is no chain.id in the data

If a user does `deployLPToken` using the wrong network, an attacker can replay the action on the correct chain, and steal the funds a-la the wintermute gnosis safe attack, where the attacker can create the same address that the user tried to, and steal the funds from there


https://mirror.xyz/0xbuidlerdao.eth/lOE5VN-BHI0olGOXe27F0auviIuoSlnou_9t3XRJseY


### Proof of Concept

```js
contracts/liquid-staking/LPTokenFactory.sol:
  26      /// @param _tokenName Name of the LP token to be deployed
  27:     function deployLPToken(
  28:         address _deployer,
  29:         address _transferHookProcessor,
  30:         string calldata _tokenSymbol,
  31:         string calldata _tokenName
  32:     ) external returns (address) {
  33:         require(address(_deployer) != address(0), "Zero address");
  34:         require(bytes(_tokenSymbol).length != 0, "Symbol cannot be zero");
  35:         require(bytes(_tokenName).length != 0, "Name cannot be zero");
  36: 
  37:         address newInstance = Clones.clone(lpTokenImplementation);
  38:         ILPTokenInit(newInstance).init(
  39:             _deployer,
  40:             _transferHookProcessor,
  41:             _tokenSymbol,
  42:             _tokenName
  43:         );
  44: 
  45:         emit LPTokenDeployed(newInstance);
  46: 
  47:         return newInstance;
  48:     }
```

### Tools Used
Manual Code Review

### Recommended Mitigation Steps
Include the chain.id 
