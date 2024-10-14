# Blockchain Writeups

## Baby ERC-20
---
For this challenge, we were given three contracts: a setup contract, an HCOIN.sol contract, and an Ownable.sol contract.
Reading the code of the setup contract:
```
// SPDX-License-Identifier: MIT
pragma solidity 0.6.12;

import { HCOIN } from "./HCOIN.sol";

contract Setup {
    HCOIN public coin;
    address player;

    constructor() public payable {
        require(msg.value == 1 ether);
        coin = new HCOIN();
        coin.deposit{value: 1 ether}();
    }

    function setPlayer(address _player) public {
      require(_player == msg.sender, "Player must be the same with the sender");
      require(_player == tx.origin, "Player must be a valid Wallet/EOA");
      player = _player;
    }

    function isSolved() public view returns (bool) {
        return coin.balanceOf(player) > 1000 ether; // im rich :D
    }
}
```
In order to solve this challenge, we need to get more than 1000 ether worth of coin tokens. 
Reading HCOIN.sol:
```
// SPDX-License-Identifier: MIT
pragma solidity 0.6.12;

import "./Ownable.sol";

contract HCOIN is Ownable {
    string public constant name = "HackerikaCoin";
    string public constant symbol = "HCOIN";
    uint8 public constant decimals = 18;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event Deposit(address indexed to, uint value);


    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        emit Deposit(msg.sender, msg.value);
    }

    function transfer(address _to, uint256 _value) public returns (bool success) {
        require(_to != address(0), "ERC20: transfer to the zero address");
        require(balanceOf[msg.sender] - _value >= 0, "Insufficient Balance");
        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;
        emit Transfer(msg.sender, _to, _value);
        return true;
    }

    function approve(address _spender, uint256 _value) public returns (bool success) {
        allowance[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    function transferFrom(address _from, address _to, uint256 _value) onlyOwner public returns (bool success) {
        require(allowance[_from][msg.sender] >= _value, "Allowance exceeded");
        require(_to != address(0), "ERC20: transfer to the zero address");
        require(balanceOf[msg.sender] - _value >= 0, "Insufficient Balance");
        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;
        allowance[_from][msg.sender] -= _value;
        emit Transfer(_from, _to, _value);
        return true;
    }

    fallback() external payable {
        deposit();
    }

}
```
Something interesting is that this contract uses Solidity 0.6.12. Solidity versions below 0.8.0 do not do checks on integer underflow or overflow, so we could potentially exploit that. In the transfer function, we can send a certain amount of tokens to another address if we have sufficient funds. However, if we pass in a number greater than our balance, it will underflow and make `balanceOf[msg.sender] - _value >= 0` true. Then, when the amount is subtracted from our balance, our balance will underflow, giving us much more than 1000 ether.
