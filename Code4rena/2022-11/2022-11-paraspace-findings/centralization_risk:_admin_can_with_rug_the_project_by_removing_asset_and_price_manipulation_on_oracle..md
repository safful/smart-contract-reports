## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-14

# [Centralization risk: admin can with rug the project by removing asset and price manipulation on oracle.](https://github.com/code-423n4/2022-11-paraspace-findings/issues/437) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/ParaSpaceOracle.sol#L21
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/configuration/ACLManager.sol#L14
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/NToken.sol#L29


# Vulnerability details

## Impact

admin can with rug the project by removing asset and price manipulation on oracle.

## Proof of Concept

As we see what happens in ankr when the admin is compromised, the centralization risk should not be taken lightly.

https://rekt.news/ankr-helio-rekt/

If the admin is compromised in paraspace, the admin can set the reward controller to invalid address, and can call resuceERC20 and rescue721 token from NToken to remove asset with no restriction or add or disable address in ACL manager directly.

```solidity
28 results - 11 files

paraspace-core\contracts\protocol\configuration\PriceOracleSentinel.sol:
  20       **/
  21:     modifier onlyPoolAdmin() {
  22          IACLManager aclManager = IACLManager(

  92          external
  93:         onlyPoolAdmin
  94      {

paraspace-core\contracts\protocol\pool\PoolConfigurator.sol:
   31       **/
   32:     modifier onlyPoolAdmin() {
   33:         _onlyPoolAdmin();
   34          _;

   91      /// @inheritdoc IPoolConfigurator
   92:     function dropReserve(address asset) external override onlyPoolAdmin {
   93          _pool.dropReserve(asset);

   99          ConfiguratorInputTypes.UpdatePTokenInput calldata input
  100:     ) external override onlyPoolAdmin {
  101          ConfiguratorLogic.executeUpdatePToken(_pool, input);

  106          ConfiguratorInputTypes.UpdateNTokenInput calldata input
  107:     ) external override onlyPoolAdmin {
  108          ConfiguratorLogic.executeUpdateNToken(_pool, input);

  113          ConfiguratorInputTypes.UpdateDebtTokenInput calldata input
  114:     ) external override onlyPoolAdmin {
  115          ConfiguratorLogic.executeUpdateVariableDebtToken(_pool, input);

  186          override
  187:         onlyPoolAdmin
  188      {

  377  
  378:     function _onlyPoolAdmin() internal view {
  379          IACLManager aclManager = IACLManager(

paraspace-core\contracts\protocol\pool\PoolParameters.sol:
   63       **/
   64:     modifier onlyPoolAdmin() {
   65:         _onlyPoolAdmin();
   66          _;

   75  
   76:     function _onlyPoolAdmin() internal view virtual {
   77          require(

  200          uint256 amountOrTokenId
  201:     ) external virtual override onlyPoolAdmin {
  202          PoolLogic.executeRescueTokens(assetType, token, to, amountOrTokenId);

paraspace-core\contracts\protocol\tokenization\DelegationAwarePToken.sol:
  33       **/
  34:     function delegateUnderlyingTo(address delegatee) external onlyPoolAdmin {
  35          IDelegationToken(_underlyingAsset).delegate(delegatee);

paraspace-core\contracts\protocol\tokenization\NToken.sol:
  130          uint256 amount
  131:     ) external override onlyPoolAdmin {
  132          IERC20(token).safeTransfer(to, amount);

  139          uint256[] calldata ids
  140:     ) external override onlyPoolAdmin {
  141          require(

  156          bytes calldata data
  157:     ) external override onlyPoolAdmin {
  158          IERC1155(token).safeBatchTransferFrom(

  170          bytes calldata airdropParams
  171:     ) external override onlyPoolAdmin {
  172          require(

paraspace-core\contracts\protocol\tokenization\NTokenApeStaking.sol:
  135  
  136:     function setUnstakeApeIncentive(uint256 incentive) external onlyPoolAdmin {
  137          ApeStakingLogic.executeSetUnstakeApeIncentive(

paraspace-core\contracts\protocol\tokenization\PToken.sol:
  333          uint256 amount
  334:     ) external override onlyPoolAdmin {
  335          require(token != _underlyingAsset, Errors.UNDERLYING_CANNOT_BE_RESCUED);

paraspace-core\contracts\protocol\tokenization\PTokenSApe.sol:
  30  
  31:     function setNToken(address _nBAYC, address _nMAYC) external onlyPoolAdmin {
  32          nBAYC = INTokenApeStaking(_nBAYC);

paraspace-core\contracts\protocol\tokenization\base\IncentivizedERC20.sol:
   26       **/
   27:     modifier onlyPoolAdmin() {
   28          IACLManager aclManager = IACLManager(

  139          external
  140:         onlyPoolAdmin
  141      {

paraspace-core\contracts\protocol\tokenization\base\MintableIncentivizedERC721.sol:
   44       **/
   45:     modifier onlyPoolAdmin() {
   46          IACLManager aclManager = IACLManager(

  132          external
  133:         onlyPoolAdmin
  134      {

  141       **/
  142:     function setBalanceLimit(uint64 limit) external onlyPoolAdmin {
  143          _ERC721Data.balanceLimit = limit;

paraspace-core\test\_pool_configurator.spec.ts:
  1199  
  1200:   it("TC-poolConfigurator-modifiers-02: Test the accessibility of onlyPoolAdmin modified functions", async () => {
  1201      const {configurator, users} = await loadFixture(testEnvFixture);
```

the admin can also manipulate the oracle by adding or removing invalid asset as collateral, set fallback oracle to invalid address that can return false data, pause or unpause the oracle with no restriction,

or just call setPrice to set a invalid price and manipulate the price directly.

```solidity
function setPrice(address _asset, uint256 _twap)
	public
	onlyRole(UPDATER_ROLE)
	onlyWhenAssetExisted(_asset)
	whenNotPaused(_asset)
{
	bool dataValidity = false;
	if (hasRole(DEFAULT_ADMIN_ROLE, msg.sender)) {
		_finalizePrice(_asset, _twap);
		return;
	}
	dataValidity = _checkValidity(_asset, _twap);
	require(dataValidity, "NFTOracle: invalid price data");
	// add price to raw feeder storage
	_addRawValue(_asset, _twap);
	uint256 medianPrice;
	// set twap price only when median value is valid
	(dataValidity, medianPrice) = _combine(_asset, _twap);
	if (dataValidity) {
		_finalizePrice(_asset, medianPrice);
	}
}
```


## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the project use multisig to safe guard private leak and also add timelock to crucial state change to avoid being attack.