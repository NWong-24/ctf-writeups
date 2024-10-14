# Blockchain Writeups

## Baby ERC-20
---
For this challenge, we were given three contracts: a setup contract, an HCOIN.sol contract, and an Ownable.sol contract.
In order to solve this challenge, we need to get more than 1000 ether worth of coin tokens. 
Something interesting is that this contract uses Solidity 0.6.12. Solidity versions below 0.8.0 do not do checks on integer underflow or overflow, so we could potentially exploit that. In the transfer function, we can send a certain amount of tokens to another address if we have sufficient funds. However, if we pass in a number greater than our balance, it will underflow and make `balanceOf[msg.sender] - _value >= 0` true. Then, when the amount is subtracted from our balance, our balance will underflow, giving us much more than 1000 ether.
