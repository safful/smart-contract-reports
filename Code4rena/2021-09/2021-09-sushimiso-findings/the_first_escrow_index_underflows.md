## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed
- resolved

# [The first escrow index underflows](https://github.com/code-423n4/2021-09-sushimiso-findings/issues/110) 

# Handle

pauliax


# Vulnerability details

## Impact
function createEscrow first assigns an index for the new isChildEscrow and only then pushes the struct to the array. When first escrow is being created, the array contains 0 elements so escrows.length-1 will underflow and return a max uint value:
   isChildEscrow[address(newEscrow)] = Fermenter(true,_templateId,escrows.length-1);
   escrows.push(newEscrow);

## Recommended Mitigation Steps
   isChildEscrow[address(newEscrow)] = Fermenter(true,_templateId,escrows.length);
   escrows.push(newEscrow);

