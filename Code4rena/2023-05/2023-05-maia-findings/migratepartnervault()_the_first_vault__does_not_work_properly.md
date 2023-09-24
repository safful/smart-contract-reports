## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-06

# [migratePartnerVault() the first vault  does not work properly](https://github.com/code-423n4/2023-05-maia-findings/issues/739) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L189


# Vulnerability details

## Impact
In the `migratePartnerVault()` method, if vaultId == 0 means illegal address, but the id of the vaults starts from 0, resulting in the first vault being mistaken as an illegal vault address

## Proof of Concept
In the `migratePartnerVault()` method, it will determine whether `newPartnerVault` is legal or not, by vaultId!=0 of vault

The code is as follows:

```solidity
    function migratePartnerVault(address newPartnerVault) external onlyOwner {
@>      if (factory.vaultIds(IBaseVault(newPartnerVault)) == 0) revert UnrecognizedVault();

        address oldPartnerVault = partnerVault;
        if (oldPartnerVault != address(0)) IBaseVault(oldPartnerVault).clearAll();
        bHermesToken.claimOutstanding();
```

But when `factory` adds vault, the index starts from 0, so the id of the first vault is 0

`PartnerManagerFactory.addVault()`
```solidity
contract PartnerManagerFactory is Ownable, IPartnerManagerFactory {
    constructor(ERC20 _bHermes, address _owner) {
        _initializeOwner(_owner);
        bHermes = _bHermes;
        partners.push(PartnerManager(address(0)));
    }

    function addVault(IBaseVault newVault) external onlyOwner {
@>      uint256 id = vaults.length;
        vaults.push(newVault);
        vaultIds[newVault] == id;

        emit AddedVault(newVault, id);
    }
```

The id of the first vault starts from 0, because in `constructor` does not add address(0) to the vaults similar to `partners`

So `migratePartnerVault()` can't be processed for the first vault


## Tools Used

## Recommended Mitigation Steps

Similar to `partners`, in the `constructor` method, a vault with address(0) is added by default. 

```solidity
contract PartnerManagerFactory is Ownable, IPartnerManagerFactory {
    constructor(ERC20 _bHermes, address _owner) {
        _initializeOwner(_owner);
        bHermes = _bHermes;
        partners.push(PartnerManager(address(0)));
+       vaults.push(IBaseVault(address(0)));
    }
```    



## Assessed type

Context