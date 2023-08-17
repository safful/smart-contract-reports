## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-06

# [RabbitHoleReceipt's address might be changed therefore only manual mint will be available](https://github.com/code-423n4/2023-01-rabbithole-findings/issues/425) 

# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L13
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L44
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L96-L118
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L95-L104
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L215-L229


# Vulnerability details

## Impact
Might be impossible to claim rewards by users. And admins must distribute tokens manually and pay fee for this. On a huge amount of participants this leads to huge amount of fees. 


## Proof of Concept

Let's consider ```QuestFactory```. It has:
```solidity
    RabbitHoleReceipt public rabbitholeReceiptContract;
```
Which responsible for mint tokens for users.

Then consider ```createQuest``` function. Here we pass ```rabbitholeReceiptContract``` into ```Quest```. 

In ```Quest``` this field is immutable.

Now lets consider next case:

1) We initialized whole contracts.
2) We created new Quest.  
3) Next we decided to change ```rabbitholeReceiptContract``` in ```QuestFactory``` for another. To do this we call: ```setRabbitHoleReceiptContract```. And successfully changing address.
4) Next we distribute signatures to our participants.
5) Users starts to mint tokens. But here is a bug ```QuestFactory``` storages new  address of ```rabbitholeReceiptContract```, but ```Quest``` initialized with older one. So users successfully minted their tokens, but can't exchange them for tokens because the Quest's receipt contract know nothing about minted tokens.

Possible solution here is change ```minterAddress``` in the original ```RabbitHoleReceipt``` contract and manually mint tokens by admin, but it will be too expensive and the company may lost a lot of money.
 

## Tools Used

Manual audit

## Recommended Mitigation Steps

In ```QuestFactory``` contract in the function ```mintReceipt```  the rabbitholeReceiptContract must be fetched from the quest directly.
To ```Quest``` Add:
```solidity
function getRabbitholeReceiptContract() public view returns(RabbitHoleReceipt) {
    return rabbitHoleReceiptContract;
}
```

Modify ```mintReceipt``` function in ```QuestFactory``` like:
```solidity
function mintReceipt(string memory questId_, bytes32 hash_, bytes memory signature_) public {
    ...
    RabbitHoleReceipt rabbitholeReceiptContract = Quest(quests[questId_].questAddress).getRabbitholeReceiptContract();
    rabbitholeReceiptContract.mint(msg.sender, questId_);
    ...
}
```