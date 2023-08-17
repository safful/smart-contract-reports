## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-09

# [Bio NFT incorrectly breaks SVG lines and doesn't support more than 120 characters effectively](https://github.com/code-423n4/2023-03-canto-identity-findings/issues/59) 

# Lines of code

https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-bio-protocol/src/Bio.sol#L43


# Vulnerability details

## Impact
Bio NFT incorrectly breaks SVG lines and doesn't support more than 120 characters effectively.

## Proof of Concept
According to the docs
> Any user can mint a Bio NFT by calling Bio.mint and passing his biography. It needs to be shorter than 200 characters.

Let's take two strings and pass them to create an NFT. The first one is 200 characters long, and the second one is 120 characters long.
`aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaW`
`aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaW`

This is how they will look like. As you can see they look identical.
![image](https://i.ibb.co/Cvw62Rz/Screenshot-from-2023-03-19-12-20-26.png)

Next, lets take this text for which we create nft. I took it from a test and double. `012345678901234567890123456789012345678👨‍👩‍👧‍👧012345678901234567890123456789012345678👨‍👩‍👧‍👧`

Here is on the left how it looks now vs how it suppose to be. As you can you line breaking doesn't work. I did enlarge
viewBox so you can see the difference.

![image](https://i.ibb.co/XDNxLWx/Screenshot-from-2023-03-19-12-28-42.png)

The problem is in this part of the code, where `(i > 0 && (i + 1) % 40 == 0)` doesn't handle properly because you want
to include emojis, so length will be more than 40 (`40 + length(emoji)`)
```solidity
canto-bio-protocol/src/Bio.sol#L56
        for (uint i; i < lengthInBytes; ++i) {
            bytes1 character = bioTextBytes[i];
            bytesLines[bytesOffset] = character;
            bytesOffset++;
            if ((i > 0 && (i + 1) % 40 == 0) || prevByteWasContinuation || i == lengthInBytes - 1) {
                bytes1 nextCharacter;
```
[canto-bio-protocol/src/Bio.sol#L56](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-bio-protocol/src/Bio.sol#L56)
Lastly, the NFT doesn't center-align text, but I believe it should. I took text from a test and on the left is how it currently appears, while on the right is how I think it should be.
 ![image](https://i.ibb.co/8r1jMhc/Screenshot-from-2023-03-18-13-43-21.png)

Here is the code. dy doesn't apply correctly; it should be 0 for the first line.
```solidity
 canto-bio-protocol/src/Bio.sol#L104
        for (uint i; i < lines; ++i) {
            text = string.concat(text, '<tspan x="50%" dy="20">', strLines[i], "</tspan>");
        }
```
[canto-bio-protocol/src/Bio.sol#L104](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-bio-protocol/src/Bio.sol#L104)
## Tools Used
Manual review
## Recommended Mitigation Steps
Enlarge viewBox so it will support 200 length or restrict to 120 characters.
Here is a complete code with correct line breaking and center text. I'm sorry that I didn't add `differ` to code because there will be too many lines. It does pass tests and fix current issues
```solidity
    function tokenURI(uint256 _id) public view override returns (string memory) {
        if (_ownerOf[_id] == address(0)) revert TokenNotMinted(_id);
        string memory bioText = bio[_id];
        bytes memory bioTextBytes = bytes(bioText);
        uint lengthInBytes = bioTextBytes.length;
        // Insert a new line after 40 characters, taking into account unicode character
        uint lines = (lengthInBytes - 1) / 40 + 1;
        string[] memory strLines = new string[](lines);
        bool prevByteWasContinuation;
        uint256 insertedLines;
        // Because we do not split on zero-width joiners, line in bytes can technically be much longer. Will be shortened to the needed length afterwards
        bytes memory bytesLines = new bytes(80);
        uint bytesOffset;
        uint j;
        for (uint i; i < lengthInBytes; ++i) {
            bytesLines[bytesOffset] = bytes1(bioTextBytes[i]);
            bytesOffset++;
            j+=1;
            if ((j>=40) || prevByteWasContinuation || i == lengthInBytes - 1) {
                bytes1 nextCharacter;
                if (i != lengthInBytes - 1) {
                    nextCharacter = bioTextBytes[i + 1];
                }
                if (nextCharacter & 0xC0 == 0x80) {
                    // Unicode continuation byte, top two bits are 10
                    prevByteWasContinuation = true;
                    continue;
                } else {
                    // Do not split when the prev. or next character is a zero width joiner. Otherwise, 👨‍👧‍👦 could become 👨>‍👧‍👦
                    // Furthermore, do not split when next character is skin tone modifier to avoid 🤦‍♂️\n🏻
                    if (
                        // Note that we do not need to check i < lengthInBytes - 4, because we assume that it's a valid UTF8 string and these prefixes imply that another byte follows
                        (nextCharacter == 0xE2 && bioTextBytes[i + 2] == 0x80 && bioTextBytes[i + 3] == 0x8D) ||
                        (nextCharacter == 0xF0 &&
                            bioTextBytes[i + 2] == 0x9F &&
                            bioTextBytes[i + 3] == 0x8F &&
                            uint8(bioTextBytes[i + 4]) >= 187 &&
                            uint8(bioTextBytes[i + 4]) <= 191) ||
                        (i >= 2 &&
                            bioTextBytes[i - 2] == 0xE2 &&
                            bioTextBytes[i - 1] == 0x80 &&
                            bioTextBytes[i] == 0x8D)
                    ) {
                        prevByteWasContinuation = true;
                        continue;
                    }
                }

                assembly {
                    mstore(bytesLines, bytesOffset)
                }
                strLines[insertedLines++] = string(bytesLines);
                bytesLines = new bytes(80);
                prevByteWasContinuation = false;
                bytesOffset = 0;
                j=0;
            }
        }
        string
            memory svg = '<svg xmlns="http://www.w3.org/2000/svg" preserveAspectRatio="xMinYMin meet" viewBox="0 0 400 100"><style>text { font-family: sans-serif; font-size: 12px; }</style>';
        string memory text = '<text x="50%" y="50%" dominant-baseline="middle" text-anchor="middle">';
        text = string.concat(text, '<tspan x="50%" dy="0">', strLines[0], "</tspan>");// center first line and than add dy
        for (uint i=1; i < lines; ++i) {
            text = string.concat(text, '<tspan x="50%" dy="20">', strLines[i], "</tspan>");
        }
        string memory json = Base64.encode(
            bytes(
                string.concat(
                    '{"name": "Bio #',
                    LibString.toString(_id),
                    '", "description": "',
                    bioText,
                    '", "image": "data:image/svg+xml;base64,',
                    Base64.encode(bytes(string.concat(svg, text, "</text></svg>"))),
                    '"}'
                )
            )
        );
        return string(abi.encodePacked("data:application/json;base64,", json));
    }

```
