## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-06

# [Incorrect emoji displaying](https://github.com/code-423n4/2023-03-canto-identity-findings/issues/185) 

# Lines of code

https://github.com/code-423n4/2023-03-canto-identity/blob/main/canto-bio-protocol/src/Bio.sol#L71-L83


# Vulnerability details

## Impact
Possible to split bio incorrectly, this leads to incorrect display of generated image. This image will be impossible to modify. This might leads to integration problems. 

## Proof of Concept
Let's consider tokenURI function of the Bio contract.
Here implemented corner case for some emojies, but current implementation doesn't handed all cases. I suppose handle all cases is redundant, erroneously, complicated and unnecessary.

Let's consider part of tokenURI code in python, this part just rewritten on Python part of Solidity code:
```python
import binascii

def sol_func_same(bio):
    bioTextBytes = str.encode(bio, "utf-8")
    lengthInBytes = len(bioTextBytes)
    lines = (lengthInBytes - 1) // 40 + 1
    strLines = [None for _ in range(lines)]
    prevByteWasContinuation = False
    insertedLines = 0
    bytesLines = []
    bytesOffset = 0
    for i in range(0, lengthInBytes):
        character = bioTextBytes[i]
        bytesLines.append(character)
        bytesOffset += 1
        if ((i > 0 and (i + 1) % 40 == 0) or prevByteWasContinuation or i == lengthInBytes - 1):
            nextCharacter = 0
            if (i != lengthInBytes - 1):  # 🏴󠁧󠁢󠁥󠁮󠁧󠁿
                nextCharacter = bioTextBytes[i + 1]
            if (nextCharacter & 0xC0 == 0x80):
                prevByteWasContinuation = True
            else:
                if (
                        (nextCharacter == 0xE2 and bioTextBytes[i + 2] == 0x80 and bioTextBytes[i + 3] == 0x8D) or
                        (nextCharacter == 0xF0 and
                         bioTextBytes[i + 2] == 0x9F and
                         bioTextBytes[i + 3] == 0x8F and
                         int(bioTextBytes[i + 4]) >= 187 and
                         int(bioTextBytes[i + 4]) <= 191) or
                        (i >= 2 and
                         bioTextBytes[i - 2] == 0xE2 and
                         bioTextBytes[i - 1] == 0x80 and
                         bioTextBytes[i] == 0x8D)
                ):
                    prevByteWasContinuation = True
                    continue

                strLines[insertedLines] = binascii.unhexlify(''.join([hex(x)[2:] for x in bytesLines])).decode("utf-8")
                insertedLines += 1
                bytesLines = []

                prevByteWasContinuation = False
                bytesOffset = 0

    for idx, i in enumerate(strLines):
        print(idx, i)


if __name__ == "__main__":
    sol_func_same("0🏴󠁧󠁢󠁥󠁮󠁧󠁿")
    print("===" * 20)
    sol_func_same("000000000000000000000000000000🏴󠁧󠁢󠁥󠁮󠁧󠁿")

```


The result is:

```
0 0🏴󠁧󠁢󠁥󠁮󠁧󠁿
============================================================
0 000000000000000000000000000000🏴󠁧󠁢
1 󠁥󠁮󠁧󠁿
``` 
In the second line present 31 character. 

but correct answer for the second case is:
```
000000000000000000000000000000🏴󠁧󠁢󠁥󠁮󠁧󠁿
```

This happened because Eng flag presented with combination of different emojies, this case doesn't handled by contract.

Similar test might be added to the Bio.t.sol contract:
```solidity

    function testSmallLine() public {
        string memory text = unicode"000🏴󠁧󠁢󠁥󠁮󠁧󠁿";
        bio.mint(text);
        string memory result = bio.tokenURI(1);
        console.logString(result);
    }

    function testLongLine() public {
        string memory text = unicode"000000000000000000000000000000🏴󠁧󠁢󠁥󠁮󠁧󠁿";
        bio.mint(text);
        string memory result = bio.tokenURI(1);
        console.logString(result);
    }
```
You need to decode them and you'll receive the same incorrect result.


## Tools Used

Manual audit, Python3 

## Recommended Mitigation Steps

Do not try to implement spec by themselves. 
1) Add view function which is responsible to create svg image. In tokenURI call this function to generate svg result.
Like this:

```solidity
function generateSvg(string memory bioText) public view returns(string) {
        bytes memory bioTextBytes = bytes(bioText);
        uint lengthInBytes = bioTextBytes.length;
        // Insert a new line after 40 characters, taking into account unicode character
        uint lines = (lengthInBytes - 1) / 40 + 1;
...
...
...
        return string(abi.encodePacked("data:application/json;base64,", json));
}
```

2) Add possibility to manual split by lines. i.e. with bio additionally store offsets in bytes, by which lines must be split.

```solidity
function mint(string calldata _bio, uint256[] calldata byteSplit) external {
...
}

function generateSvg(string memory bioText, uint256[] memory byteSplit) public view returns(string) {
...
bytes memory bioTextBytes = bytes(bioText);
in for loop:
    slice = bioTextBytes[i:i+1]
    write slice to svg
}
```

