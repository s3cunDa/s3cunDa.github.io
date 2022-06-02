# DeFi 常见标准实现整理

长期更新

## Token 类

### ERC20

最广泛使用的 Token。

### ERC721

常见的 NFT 类型实现，每个 NFT 由唯一的 tokenId标识。

由 \_owners(uint256 => address) 保存 token 的 所有关系，\_balances(address => uint256) 保存地址拥有 NFT 数量。

\_tokenApprovals(uint256 => address) 记录 NFT 的委托权限，通过 \_operatorApprovals(address => mapping(address =>bool)) 记录地址所有 NFT 委托给 operator 。

基本外部调用接口为：

```
interface IERC721 is IERC165 {
    function balanceOf(address owner) external view returns (uint256 balance);
    function ownerOf(uint256 tokenId) external view returns (address owner);
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes calldata data
    ) external;
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId
    ) external;
    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) external;
    function approve(address to, uint256 tokenId) external;
    function setApprovalForAll(address operator, bool _approved) external;
    function getApproved(uint256 tokenId) external view returns (address operator);
    function isApprovedForAll(address owner, address operator) external view returns (bool);
}
```

内置了\_checkOnERC721Received 回调函数，如果接收方是合约，则需要实现 onERC721Received 函数来处理接收逻辑，可能有重入的风险。

注意，由于 NFT 是以 tokenId 作为标识进行的 approve，所以在 NFT 交易过后需要对之前的 approve 信息进行清除，OpenZepplin 实现了这一点：

```
    function _transfer(
        address from,
        address to,
        uint256 tokenId
    ) internal virtual {
        require(ERC721.ownerOf(tokenId) == from, "ERC721: transfer from incorrect owner");
        require(to != address(0), "ERC721: transfer to the zero address");

        _beforeTokenTransfer(from, to, tokenId);

        // Clear approvals from the previous owner
        _approve(address(0), tokenId);

        _balances[from] -= 1;
        _balances[to] += 1;
        _owners[tokenId] = to;

        emit Transfer(from, to, tokenId);

        _afterTokenTransfer(from, to, tokenId);
    }
```

### ERC777

ERC20 的增强版，主要做的改进就是在转账和收款的时候进行了 hook，会调用交易对方的回调函数进行条件判断或者其他逻辑。

为了和 ERC20 进行区分以及兼容，将 ERC20 的 transfer 和 transferFrom 保留，ERC777 相关的转账操作由 send 接口来完成。

ERC777 新增了 operator 角色，operator 可以对 holder 的所有 ERC777 token 进行管理，当然，holder 是自己的 operator。

可以当作 ERC20 使用，如果使用 ERC777 进行转账时，转账前会调用 from.tokensToSend(operator, from, to, amount, userData, operatorData)，转账后会调用 to.tokensReceived(operator, from, to, amount, userData, operatorData)。

主要接口如下：

```
interface ERC777Token {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function totalSupply() external view returns (uint256);
    function balanceOf(address holder) external view returns (uint256);
    function granularity() external view returns (uint256);

    function defaultOperators() external view returns (address[] memory);
    function isOperatorFor(
        address operator,
        address holder
    ) external view returns (bool);
    function authorizeOperator(address operator) external;
    function revokeOperator(address operator) external;

    function send(address to, uint256 amount, bytes calldata data) external;
    function operatorSend(
        address from,
        address to,
        uint256 amount,
        bytes calldata data,
        bytes calldata operatorData
    ) external;

    function burn(uint256 amount, bytes calldata data) external;
    function operatorBurn(
        address from,
        uint256 amount,
        bytes calldata data,
        bytes calldata operatorData
    ) external;
}
```

### ERC1155

ERC1155 是对 ERC20 和 ERC721 这两种代币的融合，ERC1155 可以表示 NFT 或者 ERC20，并且允许二者打包操作。

由于将 NFT 与 ERC20 混合在一起，所以 balance 的表示形式为 (*uint256* => *mapping*(*address* => *uint256*)) 即 id -> address -> balance

approve 的形式也是直接设置 operator 来进行管理。

每个 id 对应的 token 可以为 NFT 或者同质化货币，所以在铸币时需要指明生成的 id 以及数量。

与 ERC777 类似，在转账后都会调用收款方的 onERC1155Received 回调函数。

可以理解为这是一个大的 ERC20 集合，用id来表明各个不同的 ERC20，发行量决定是否是 NFT（1个就是NFT）。

```
interface ERC777Token {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function totalSupply() external view returns (uint256);
    function balanceOf(address holder) external view returns (uint256);
    function granularity() external view returns (uint256);

    function defaultOperators() external view returns (address[] memory);
    function isOperatorFor(
        address operator,
        address holder
    ) external view returns (bool);
    function authorizeOperator(address operator) external;
    function revokeOperator(address operator) external;

    function send(address to, uint256 amount, bytes calldata data) external;
    function operatorSend(
        address from,
        address to,
        uint256 amount,
        bytes calldata data,
        bytes calldata operatorData
    ) external;

    function burn(uint256 amount, bytes calldata data) external;
    function operatorBurn(
        address from,
        uint256 amount,
        bytes calldata data,
        bytes calldata operatorData
    ) external;
}
```

## 权限管理

### AccessControl

代码：https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/AccessControl.sol

常用的权限管理库，主要思路就是维护一个 (*bytes32* => RoleData) 的字典，bytes32是角色名，RoleData结构如下：

```
    struct RoleData {
        mapping(address => bool) members;
        bytes32 adminRole;
    }
```

其中 adminRole为可以管理当前 role 的身份。 

需要进行权限检查时，则只需要加修饰符 *modifier* onlyRole(*bytes32* role) 进行检查即可。

### Ownable

代码：https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable.sol

AccessControl的简化版本，只对owner进行相关的检查。

## 代理

代理的主要思路都是代码和存储隔离，并且保证代码的可升级，也就是 implementation 的更改。

目前主要有两种实现模式，即 UUPS 以及 transparent。

UUPS 实现的形式是将升级的逻辑放进 implementation 中，而 transparent 则是将升级的逻辑都写在 proxy 中。

推荐使用 UUPS，一方面是 gas 友好（虽然不知道为什么），另一方面是 transparent 中内置的函数或许会和 code 的函数签名造成冲突，可能会出一些问题。

## 多签 & 时间锁

### gnosis multisig

### timelock

## DEX

### Uniswap 类

