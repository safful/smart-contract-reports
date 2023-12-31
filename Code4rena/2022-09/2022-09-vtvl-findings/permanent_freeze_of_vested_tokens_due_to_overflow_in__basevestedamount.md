## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [Permanent freeze of vested tokens due to overflow in _baseVestedAmount](https://github.com/code-423n4/2022-09-vtvl-findings/issues/95) 

# Lines of code

https://github.com/code-423n4/2022-09-vtvl/blob/f68b7f3e61dad0d873b5b5a1e8126b839afeab5f/contracts/VTVLVesting.sol#L176


# Vulnerability details

## Description 
The _baseVestedAmount() function calculates vested amount for some (claim, timestamp) pair. It is wrapped by several functions, like vestedAmount, which is used in withdraw() to calculate how much a user can retrieve from their claim. Therefore, it is critical that this function will calculate correctly for users to receive their funds.

Below is the calculation of the linear vest amount:
```
uint112 linearVestAmount = _claim.linearVestAmount * truncatedCurrentVestingDurationSecs / finalVestingDurationSecs;
```
Importantly, _claim.linearVestAmount is of type uint112 and truncatedCurrentVestingDurationSecs is of type uint40. Using compiler >= 0.8.0,  the product cannot exceed uint112 or else the function reverts due to overflow. In fact, we can show that uint112 is an inadequate size for this calculation.

The max value for uint112 is 5192296858534827628530496329220096. 
Seconds in year = 3600 * 24 * 365 = 31536000 
Tokens that inherit from ERC20 like the ones used in VTVL have 18 decimal places -> 1000000000000000000
This means the maximum number of tokens that are safe to vest for one year is 2**112 / 10e18 / (3600 * 24 * 365) = just 16,464,665 tokens. 
This is definitely not a very large amount and it is expected that some projects will mint a similar or larger amount for vesting for founders / early employees. For 4 year vesting, the safe amount drops to 4,116,166.
Projects that are not forewarned about this size limit are likely to suffer from freeze of funds of employees, which will require very patchy manual revocation and restructuring of the vesting to not overflow.

## Impact
Employees/founders do not have access to their vested tokens.

## Proof of Concept
Below is a test that demonstrates the overflow issue, 1 year after 17,000,000 tokens have matured.
```
describe('Long vest fail', async () => {
  let vestingContract: VestingContractType;
  // Default params
  // linearly Vest 10000, every 1s, between TS 1000 and 2000
  // additionally, cliff vests another 5000, at TS = 900
  const recipientAddress = await randomAddress();
  const startTimestamp = BigNumber.from(1000);
  const endTimestamp = BigNumber.from(1000 + 3600 * 24 * 365);
  const midTimestamp = BigNumber.from(1000 + (3600 * 24 * 365) / 2);
  const cliffReleaseTimestamp = BigNumber.from(0);
  const linearVestAmount = BigNumber.from('170000000000000000000000000');
  const cliffAmount = BigNumber.from(0);
  const releaseIntervalSecs = BigNumber.from(5);

  before(async () => {
    const {vestingContract: _vc} = await createPrefundedVestingContract({tokenName, tokenSymbol, initialSupplyTokens});
    vestingContract = _vc;
    await vestingContract.createClaim(recipientAddress, startTimestamp, endTimestamp, cliffReleaseTimestamp, releaseIntervalSecs, linearVestAmount, cliffAmount);
  });

  it('half term works', async() => {
    expect(await vestingContract.vestedAmount(recipientAddress, midTimestamp)).to.be.equal('85000000000000000000000000');
  });

  it('full term fails', async() => {
    // Note: at exactly the cliff time, linear vested amount won't yet come in play as we're only at second 0
    await expect(vestingContract.vestedAmount(recipientAddress, endTimestamp)).to.be.revertedWithPanic(0x11
    );
  });
});
```

## Tools Used
Manual audit, hardhat / chai.

## Recommended Mitigation Steps
Perform the intermediate calculation of linearVestAmount using the uint256 type.
```
uint112 linearVestAmount = uint112( uint256(_claim.linearVestAmount) * truncatedCurrentVestingDurationSecs / finalVestingDurationSecs);
```
