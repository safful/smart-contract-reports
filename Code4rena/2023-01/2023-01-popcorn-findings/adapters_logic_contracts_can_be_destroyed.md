## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- edited-by-warden
- M-05

# [Adapters logic contracts can be destroyed](https://github.com/code-423n4/2023-01-popcorn-findings/issues/700) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/dcdd3ceda3d5bd87105e691ebc054fb8b04ae583/src/vault/adapter/abstracts/AdapterBase.sol#L444-L446
https://github.com/code-423n4/2023-01-popcorn/blob/dcdd3ceda3d5bd87105e691ebc054fb8b04ae583/src/vault/adapter/abstracts/AdapterBase.sol#L55-L62
https://github.com/code-423n4/2023-01-popcorn/blob/36477d96788791ff07a1ba40d0c726fb39bf05ec/src/vault/adapter/beefy/BeefyAdapter.sol#L20-L55
https://github.com/code-423n4/2023-01-popcorn/blob/36477d96788791ff07a1ba40d0c726fb39bf05ec/src/vault/adapter/yearn/YearnAdapter.sol#L17-L43


# Vulnerability details

## Impact

AdapterBase based adapters instances like BeefyAdapter, and YearnAdapter can be rendered inoperable and halt the project.

## Proof of Concept

When using upgradeable smart contracts, all interactions occur with the contract instance, not the underlying logic contract.
A malicious actor sending transactions directly to the logic contract  does not pose a significant threat because changes made to the state of the logic contract will not affect the contract instance as the logic contract's storage is never utilized.

However, there is an exception to this rule. If a direct call to the logic contract results in a self-destruct operation, the logic contract will be eliminated, and all instances of your contract will delegate calls to a code-less address rendering all contract instances in the project inoperable.

Similarly, if the logic contract contains a delegatecall operation and is made to delegate a call to a malicious contract with a self-destruct function, the calling contract will also be destroyed.

The AdapterBase contract has an internal initializer function [__AdapterBase_init](https://github.com/code-423n4/2023-01-popcorn/blob/dcdd3ceda3d5bd87105e691ebc054fb8b04ae583/src/vault/adapter/abstracts/AdapterBase.sol#L55-L79) that among the other things, allows to assign the strategy address (see [line 62](https://github.com/code-423n4/2023-01-popcorn/blob/dcdd3ceda3d5bd87105e691ebc054fb8b04ae583/src/vault/adapter/abstracts/AdapterBase.sol#L62), and [line 79](https://github.com/code-423n4/2023-01-popcorn/blob/dcdd3ceda3d5bd87105e691ebc054fb8b04ae583/src/vault/adapter/abstracts/AdapterBase.sol#L79)).

[AdapterBase](https://github.com/code-423n4/2023-01-popcorn/blob/dcdd3ceda3d5bd87105e691ebc054fb8b04ae583/src/vault/adapter/abstracts/AdapterBase.sol#L55-L79) excerpt:
```solidity
    function __AdapterBase_init(bytes memory popERC4626InitData)
        internal
        onlyInitializing
    {
        (
            address asset,
            address _owner,
            address _strategy,
            uint256 _harvestCooldown,
            bytes4[8] memory _requiredSigs,
            bytes memory _strategyConfig
        ) = abi.decode(
                popERC4626InitData,
                (address, address, address, uint256, bytes4[8], bytes)
            );
        __Owned_init(_owner);
        __Pausable_init();
        __ERC4626_init(IERC20Metadata(asset));

        INITIAL_CHAIN_ID = block.chainid;
        INITIAL_DOMAIN_SEPARATOR = computeDomainSeparator();

        _decimals = IERC20Metadata(asset).decimals();

        strategy = IStrategy(_strategy);
```

The function harvest does a delegatecall to the address defined by strategy:

```solidity
    function harvest() public takeFees {
        if (
            address(strategy) != address(0) &&
            ((lastHarvest + harvestCooldown) < block.timestamp)
        ) {
            // solhint-disable
            address(strategy).delegatecall(
                abi.encodeWithSignature("harvest()")
            );
        }

        emit Harvested();
    }
```

An attacker can call the initializer method of the BeefyAdapter / YearnAdapter to pass a malicious contract to initialize the strategy address of AdaptereBase, where the malicious contract only has a harvest() function or a fallback function that calls selfdestruct.
The attacker will then call harvest on BeefyAdapter / YearnAdapter implementation causing the logic contracts to be destroyed.

[YearnAdapter](https://github.com/code-423n4/2023-01-popcorn/blob/36477d96788791ff07a1ba40d0c726fb39bf05ec/src/vault/adapter/yearn/YearnAdapter.sol#L17-L43) excerpt:
```solidity
contract YearnAdapter is AdapterBase {
    using SafeERC20 for IERC20;
    using Math for uint256;

    string internal _name;
    string internal _symbol;

    VaultAPI public yVault;
    uint256 constant DEGRADATION_COEFFICIENT = 10**18;

    /**
     * @notice Initialize a new Yearn Adapter.
     * @param adapterInitData Encoded data for the base adapter initialization.
     * @param externalRegistry Yearn registry address.
     * @dev This function is called by the factory contract when deploying a new vault.
     * @dev The yearn registry will be used given the `asset` from `adapterInitData` to find the latest yVault.
     */
    function initialize(
        bytes memory adapterInitData,
        address externalRegistry,
        bytes memory
    ) external initializer {
        (address _asset, , , , , ) = abi.decode(
            adapterInitData,
            (address, address, address, uint256, bytes4[8], bytes)
        );
        __AdapterBase_init(adapterInitData);
```

[BeefyAdapter](https://github.com/code-423n4/2023-01-popcorn/blob/36477d96788791ff07a1ba40d0c726fb39bf05ec/src/vault/adapter/beefy/BeefyAdapter.sol#L20-L55) excerpt:
```solidity
contract BeefyAdapter is AdapterBase, WithRewards {
    using SafeERC20 for IERC20;
    using Math for uint256;

    string internal _name;
    string internal _symbol;

    IBeefyVault public beefyVault;
    IBeefyBooster public beefyBooster;
    IBeefyBalanceCheck public beefyBalanceCheck;

    uint256 public constant BPS_DENOMINATOR = 10_000;

    error NotEndorsed(address beefyVault);
    error InvalidBeefyVault(address beefyVault);
    error InvalidBeefyBooster(address beefyBooster);

    /**
     * @notice Initialize a new Beefy Adapter.
     * @param adapterInitData Encoded data for the base adapter initialization.
     * @param registry Endorsement Registry to check if the beefy adapter is endorsed.
     * @param beefyInitData Encoded data for the beefy adapter initialization.
     * @dev `_beefyVault` - The underlying beefy vault.
     * @dev `_beefyBooster` - An optional beefy booster.
     * @dev This function is called by the factory contract when deploying a new vault.
     */
    function initialize(
        bytes memory adapterInitData,
        address registry,
        bytes memory beefyInitData
    ) external initializer {
        (address _beefyVault, address _beefyBooster) = abi.decode(
            beefyInitData,
            (address, address)
        );
        __AdapterBase_init(adapterInitData);
```
## Tools Used

## Recommended Mitigation Steps

Add constructors and use OZ Initializable's _disableInitializers() function as the only line of code in the constructor.

```solidity
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }
```

Also suggest the same for MultiRewardStaking and Vault contracts.