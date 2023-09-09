## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor confirmed
- H-02

# [BeefyAdapter() malicious vault owner can use malicious _beefyBooster to steal the adapter's token](https://github.com/code-423n4/2023-01-popcorn-findings/issues/552) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/adapter/beefy/BeefyAdapter.sol#L61-L65


# Vulnerability details

## Impact
malicious vault owner can use Malicious _beefyBooster to steal the adapter's token

## Proof of Concept

When creating a BeefyAdapter, the vault owner can specify the _beefyBooster
The current implementation does not check if the _beefyBooster is legitimate or not, and worse, it _beefyVault.approve to the _beefyBooster during initialization
The code is as follows:

```solidity
contract BeefyAdapter is AdapterBase, WithRewards {
...
    function initialize(
        bytes memory adapterInitData,
        address registry,
        bytes memory beefyInitData
    ) external initializer {
       
        (address _beefyVault, address _beefyBooster) = abi.decode(
            beefyInitData,   //@audit <--------- beefyInitData comes from the owner's input: adapterData.data
            (address, address)
        );


        //@audit <-------- not check _beefyBooster is legal
        if (
            _beefyBooster != address(0) &&
            IBeefyBooster(_beefyBooster).stakedToken() != _beefyVault  
        ) revert InvalidBeefyBooster(_beefyBooster);   

...
      
        if (_beefyBooster != address(0))
            IERC20(_beefyVault).approve(_beefyBooster, type(uint256).max);     //@audit <---------  _beefyVault approve _beefyBooster    

}

    function _protocolDeposit(uint256 amount, uint256)
        internal
        virtual
        override
    {
        beefyVault.deposit(amount);
        if (address(beefyBooster) != address(0)) 
            beefyBooster.stake(beefyVault.balanceOf(address(this)));  //@audit <--------- A malicious beefyBooster can transfer the token
    }
```

As a result, a malicious user can pass a malicious _beefyBooster contract, and when the user deposits to the vault, the vault is saved to the _beefyVault,
This malicious _beefyBooster can execute _beefyVault.transferFrom(BeefyAdapter), and take all the tokens stored by the adapter to _beefyVault

## Tools Used

## Recommended Mitigation Steps

Check _beefyBooster just like check _beefyVault

```solidity
    function initialize(
        bytes memory adapterInitData,
        address registry,
        bytes memory beefyInitData
    ) external initializer {
...
        if (!IPermissionRegistry(registry).endorsed(_beefyVault))
            revert NotEndorsed(_beefyVault);
...            
+       if (!IPermissionRegistry(registry).endorsed(_beefyBooster))
+           revert NotEndorsed(_beefyBooster);

        if (
            _beefyBooster != address(0) &&
            IBeefyBooster(_beefyBooster).stakedToken() != _beefyVault
        ) revert InvalidBeefyBooster(_beefyBooster);
```