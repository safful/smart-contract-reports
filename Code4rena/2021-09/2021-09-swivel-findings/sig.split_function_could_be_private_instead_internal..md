## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Sig.split function could be private instead internal.](https://github.com/code-423n4/2021-09-swivel-findings/issues/44) 

# Handle

pants


# Vulnerability details

The Sig.split function (defined at Line 40 of Sig library) is called only inside the library. Thus function could be set as private.

https://github.com/Swivel-Finance/gost/blob/v2/test/swivel/Sig.sol#L40

## Recommended Mitigation Steps
Make it private :)

## Tool Used
Manual code review.


