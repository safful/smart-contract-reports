## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- M-04

# [Bio Protocol - `tokenURI` JSON injection](https://github.com/code-423n4/2023-03-canto-identity-findings/issues/212) 

# Lines of code

https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-bio-protocol/src/Bio.sol#L103-L115


# Vulnerability details

## Impact
The Bio Protocol allows users to mint Bio NFTs that represent user's bio. Once NFT is minted anyone can trigger `tokenURI` to retrieve JSON data with the bio and generated svg image. Example JSON content (decoded from Base64):
```
{"name": "Bio #1", "description": "Test", "image": "data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHByZXNlcnZlQXNwZWN0UmF0aW89InhNaW5ZTWluIG1lZXQiIHZpZXdCb3g9IjAgMCA0MDAgMTAwIj48c3R5bGU+dGV4dCB7IGZvbnQtZmFtaWx5OiBzYW5zLXNlcmlmOyBmb250LXNpemU6IDEycHg7IH08L3N0eWxlPjx0ZXh0IHg9IjUwJSIgeT0iNTAlIiBkb21pbmFudC1iYXNlbGluZT0ibWlkZGxlIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIj48dHNwYW4geD0iNTAlIiBkeT0iMjAiPlRlc3Q8L3RzcGFuPjwvdGV4dD48L3N2Zz4="}
```

The issue is that function `Bio.tokenURI` is vulnerable to JSON injection and it is possible to inject `"` character into bio completely altering the integrity of JSON data. User sends following data as `bio`:
```
Test\", \"name\": \"Bio #999999999
```
This is will result in following JSON data (new `name` key was injected):
```
{"name": "Bio #1", "description": "Test", "name": "Bio #999999999", "image": "data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHByZXNlcnZlQXNwZWN0UmF0aW89InhNaW5ZTWluIG1lZXQiIHZpZXdCb3g9IjAgMCA0MDAgMTAwIj48c3R5bGU+dGV4dCB7IGZvbnQtZmFtaWx5OiBzYW5zLXNlcmlmOyBmb250LXNpemU6IDEycHg7IH08L3N0eWxlPjx0ZXh0IHg9IjUwJSIgeT0iNTAlIiBkb21pbmFudC1iYXNlbGluZT0ibWlkZGxlIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIj48dHNwYW4geD0iNTAlIiBkeT0iMjAiPlRlc3QiLCAibmFtZSI6ICJCaW8gIzk5OTk5OTk5OTwvdHNwYW4+PC90ZXh0Pjwvc3ZnPg=="}
```

Depending on the usage of Bio Protocol by frontend or 3rd party applications this will have serious impact on the security. Some examples of the attacks that can be carried:
1. Attacker might create a Bio NFT that mimics copies the name of different Bio NFT.
2. Attacker might change the generated svg `image` to completely different one.
3. Attacker might point `image` to external `URL`.
4. Attacker might trigger cross-site scripting attacks if the data from JSON is not properly handled.

## Proof of Concept
https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-bio-protocol/src/Bio.sol#L103-L115


### Tools Used
Manual Review / Foundry / VSCode

## Recommended Mitigation Steps
It is recommended to properly encode user's bio to make sure it cannot alter returned JSON data.