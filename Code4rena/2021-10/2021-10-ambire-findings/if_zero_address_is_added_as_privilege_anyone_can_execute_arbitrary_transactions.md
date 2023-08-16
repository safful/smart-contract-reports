## Tags

- bug
- sponsor confirmed
- 1 (Low Risk)
- resolved

# [If zero address is added as privilege anyone can execute arbitrary transactions](https://github.com/code-423n4/2021-10-ambire-findings/issues/37) 

# Handle

cmichel


# Vulnerability details

The `SignatureValidator.recoverAddrImpl` function does not revert on invalid signatures and returns zero instead.
Thus if anyone added the zero address to their `privileges` by accident, funds can be stolen in `Identity.execute`.

## Recommended Mitigation Steps
Unless there's a valid reason for the `SignatureMode.NoSig` mode, consider reverting if `ecrecover` returns the zero address indicating an invalid signature.


