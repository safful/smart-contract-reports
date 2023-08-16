## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [missing whenNotPaused](https://github.com/code-423n4/2022-01-sherlock-findings/issues/280) 

# Handle

danb


# Vulnerability details

all external function which are not onlyOwner have whenNotPaused modifier.
but transfer doesn't have it (Sherlok is ERC721).
https://github.com/code-423n4/2022-01-sherlock/blob/main/contracts/Sherlock.sol#L366

this was very important during the exploit to badger dao:
https://mobile.twitter.com/flashfish0x/status/1466369783016869892

i suggest you add whenNotPaused to _beforeTokenTransfer



