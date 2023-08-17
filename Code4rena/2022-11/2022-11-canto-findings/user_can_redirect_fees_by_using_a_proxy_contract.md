## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor acknowledged
- edited-by-warden
- H-01

# [User can redirect fees by using a proxy contract](https://github.com/code-423n4/2022-11-canto-findings/issues/48) 

# Lines of code

https://github.com/code-423n4/2022-11-canto/blob/main/CIP-001/src/Turnstile.sol#L86-L101
https://github.com/code-423n4/2022-11-canto/blob/main/Canto/x/csr/keeper/evm_hooks.go#L51


# Vulnerability details

## Impact
For any given tx, the fees are sent to its recipient (`To`). Anybody can register an address using the Turnstile contract. Thus, a user is able to create a proxy contract with which they execute other smart contracts. That way, the fees are sent to their own contract instead of the actual application they are using. People who use smart contract wallets don't even have to bother with setting up a proxy structure. They just add their own wallet to the Turnstile contract.

Also, there might be a possibility of someone setting up a proxy for high-usage contracts where the fees are sent back to the caller. So for contract $X$, we create $X'$ which calls $X$ for the caller. Since $X'$ is the recipient of the tx, it gets the gas refund. To incentivize the user to use $X'$ instead of $X$, $X'$ sends a percentage of the refund to the caller. The feasibility both technically and economically depends on the contract that is attacked. But, theoretically, it's possible.

The incentive to take it for yourself instead of giving it to the app is pretty high. Since this causes a loss of funds for the app I rate it as HIGH. 

## Proof of Concept
Registering an address is permissionless:
```sol
    function register(address _recipient) public onlyUnregistered returns (uint256 tokenId) {
        address smartContract = msg.sender;

        if (_recipient == address(0)) revert InvalidRecipient();

        tokenId = _tokenIdTracker.current();
        _mint(_recipient, tokenId);
        _tokenIdTracker.increment();

        emit Register(smartContract, _recipient, tokenId);

        feeRecipient[smartContract] = NftData({
            tokenId: tokenId,
            registered: true
        });
    }
```

Fees are sent to the recipient of the tx:
```sol
func (h Hooks) PostTxProcessing(ctx sdk.Context, msg core.Message, receipt *ethtypes.Receipt) error {
	// Check if the csr module has been enabled
	params := h.k.GetParams(ctx)
	if !params.EnableCsr {
		return nil
	}

	// Check and process turnstile events if applicable
	h.processEvents(ctx, receipt)

	contract := msg.To()
	if contract == nil {
		return nil
	}

        // ...
```

## Tools Used
none

## Recommended Mitigation Steps
It's pretty difficult to fix this properly. The ideal solution is to distribute fees according to each contract's gas usage. That will be a little more complicated to implement. Also, you have to keep an eye on whether it incentivizes developers to make their contracts less efficient. Another solution is to make this feature permissioned so that only select contracts are allowed to participate. For example, you could say that an address has to be triggered $X$ amount of times before it is eligible for gas refunds.
