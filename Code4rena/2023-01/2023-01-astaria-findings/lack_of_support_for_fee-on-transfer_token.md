## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-33

# [Lack of support for fee-on-transfer token](https://github.com/code-423n4/2023-01-astaria-findings/issues/51) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/TransferProxy.sol#L34
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L181
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L643


# Vulnerability details

## Impact
\
Lack of support for fee-on-transfer token.

## Proof of Concept

In the codebase, the usage of safeTransfer and safeTransferFrom assume that the receiver receive the exact transferred amount.

```solidity
src\AstariaRouter.sol:
  528      ERC20(IAstariaVaultBase(commitments[0].lienRequest.strategy.vault).asset())
  529:       .safeTransfer(msg.sender, totalBorrowed);
  530    }

src\ClearingHouse.sol:
  142  
  143:     ERC20(paymentToken).safeTransfer(
  144        s.auctionStack.liquidator,

  160      if (ERC20(paymentToken).balanceOf(address(this)) > 0) {
  161:       ERC20(paymentToken).safeTransfer(
  162          ASTARIA_ROUTER.COLLATERAL_TOKEN().ownerOf(collateralId),

src\PublicVault.sol:
  383  
  384:       ERC20(asset()).safeTransfer(currentWithdrawProxy, withdrawBalance);
  385        WithdrawProxy(currentWithdrawProxy).increaseWithdrawReserveReceived(

src\VaultImplementation.sol:
  393      payout = _handleProtocolFee(c.lienRequest.amount);
  394:     ERC20(asset()).safeTransfer(receiver, payout);
  395    }

  405        }
  406:       ERC20(asset()).safeTransfer(feeTo, fee);
  407      }

src\WithdrawProxy.sol:
  268      if (s.withdrawRatio == uint256(0)) {
  269:       ERC20(asset()).safeTransfer(VAULT(), balance);
  270      } else {

  280        if (balance > 0) {
  281:         ERC20(asset()).safeTransfer(VAULT(), balance);
  282        }

  297      }
  298:     ERC20(asset()).safeTransfer(withdrawProxy, amount);
  299      return amount;
```

However, according to https://github.com/d-xo/weird-erc20#fee-on-transfer

Some tokens take a transfer fee (e.g. STA, PAXG), some do not currently charge a fee but may do so in the future (e.g. USDT, USDC).

So the recipient address may not receive the full transfered amount, which can break the protocol's accounting and revert transaction.

The same safeTransfer and safeTransferFrom is used in the vault deposit / withdraw / mint / withdraw function.

Let us see a concrete example,

```solidity
contract TransferProxy is Auth, ITransferProxy {
  using SafeTransferLib for ERC20;

  constructor(Authority _AUTHORITY) Auth(msg.sender, _AUTHORITY) {
    //only constructor we care about is  Auth
  }

  function tokenTransferFrom(
    address token,
    address from,
    address to,
    uint256 amount
  ) external requiresAuth {
    ERC20(token).safeTransferFrom(from, to, amount);
  }
}
```

the transfer Proxy also use

```solidity
ERC20(token).safeTransferFrom(from, to, amount);
```

this transfer is used extensively

```solidity

src\AstariaRouter.sol:
  208      RouterStorage storage s = _loadRouterSlot();
  209:     s.TRANSFER_PROXY.tokenTransferFrom(
  210        address(token),

src\LienToken.sol:
  184      );
  185:     s.TRANSFER_PROXY.tokenTransferFrom(
  186        params.encumber.stack[params.position].lien.token,

  654      if (payment > 0)
  655:       s.TRANSFER_PROXY.tokenTransferFrom(token, payer, payee, payment);
  656  

  860  
  861:     s.TRANSFER_PROXY.tokenTransferFrom(stack.lien.token, payer, payee, amount);
  862  

src\scripts\deployments\Deploy.sol:
  378        uint8(UserRoles.ASTARIA_ROUTER),
  379:       TRANSFER_PROXY.tokenTransferFrom.selector,
  380        true

  403        uint8(UserRoles.LIEN_TOKEN),
  404:       TRANSFER_PROXY.tokenTransferFrom.selector,
  405        true
```

If the token used charge transfer fee, the accounting below is broken:

When _payDebt is called

```solidity
  function _payDebt(
    LienStorage storage s,
    address token,
    uint256 payment,
    address payer,
    AuctionStack[] memory stack
  ) internal returns (uint256 totalSpent) {
    uint256 i;
    for (; i < stack.length;) {
      uint256 spent;
      unchecked {
        spent = _paymentAH(s, token, stack, i, payment, payer);
        totalSpent += spent;
        payment -= spent;
        ++i;
      }
    }
  }
```

which calls _paymentAH

```solidity
  function _paymentAH(
    LienStorage storage s,
    address token,
    AuctionStack[] memory stack,
    uint256 position,
    uint256 payment,
    address payer
  ) internal returns (uint256) {
    uint256 lienId = stack[position].lienId;
    uint256 end = stack[position].end;
    uint256 owing = stack[position].amountOwed;
    //checks the lien exists
    address payee = _getPayee(s, lienId);
    uint256 remaining = 0;
    if (owing > payment.safeCastTo88()) {
      remaining = owing - payment;
    } else {
      payment = owing;
    }
    if (payment > 0)
      s.TRANSFER_PROXY.tokenTransferFrom(token, payer, payee, payment);

    delete s.lienMeta[lienId]; //full delete
    delete stack[position];
    _burn(lienId);

    if (_isPublicVault(s, payee)) {
      IPublicVault(payee).updateAfterLiquidationPayment(
        IPublicVault.LiquidationPaymentParams({remaining: remaining})
      );
    }
    emit Payment(lienId, payment);
    return payment;
  }
```

not that the code

```solidity
s.TRANSFER_PROXY.tokenTransferFrom(token, payer, payee, payment);
```

if the token charge transfer fee, for example, the payment amout is 100 ETH, 1 ETH is charged as fee, the recipient only receive 99 ETH,

but the wrong value payment 100 ETH is returned and used to update the accounting

```solidity
unchecked {
	spent = _paymentAH(s, token, stack, i, payment, payer);
	totalSpent += spent;
	payment -= spent;
  ++i;
}
```

then the variable totalSpent and payment amoutn will be not valid.

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the protocol whitelist the token address or use balance before and after check to make sure the recipient receive the accurate amount of token when token transfer is performed.