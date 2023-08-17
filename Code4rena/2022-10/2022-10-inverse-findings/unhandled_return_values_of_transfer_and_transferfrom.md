## Tags

- bug
- 2 (Med Risk)
- satisfactory
- sponsor acknowledged
- selected for report
- M-01

# [Unhandled return values of transfer and transferFrom](https://github.com/code-423n4/2022-10-inverse-findings/issues/10) 

# Lines of code

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L205
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L280
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L399
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L537
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L570
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L602


# Vulnerability details

## Impact
ERC20 implementations are not always consistent. Some implementations of transfer and transferFrom could return ‘false’ on failure instead of reverting. It is safer to wrap such calls into require() statements to these failures.


## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L205
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L280
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L399
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L537
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L570
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L602
## Tools Used
Read the codes

## Recommended Mitigation Steps
Check the return value and revert on 0/false or use OpenZeppelin’s SafeERC20 wrapper functions