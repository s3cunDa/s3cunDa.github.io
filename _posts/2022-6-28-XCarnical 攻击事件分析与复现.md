# XCarnical 攻击事件分析与复现

## 事件概述

2022年6月27日，部署在 ETH 主网上 NFT 抵押借贷类 DeFi 项目，遭到攻击，项目方损失约380万美元。

XCarnical 支持 BAYC 和 CryptoPunk 系列的 NFT 作为抵押物贷出 ETH。在用户向协议中抵押 NFT 并进行借款时，XCarnical 没有对用户抵押信息的 isWithdraw 字段进行检查，造成用户可以在抵押 NFT 并取回后继续进行借贷操作，攻击者以此获利。

事件发生后，XCarnival 项目方雨攻击者进行链上交涉，最终攻击者同意将一半的被盗资金返还给项目方，整个事件到此结束。

## 漏洞分析

XNFT.pledge 为用户使用 NFT 进行抵押的接口函数，用户向协议转入 NFT，协议生成一个 order 结构体存储用户的抵押借贷信息。Order 结构体中存储了用户抵押的 NFT 信息以及一个标注用户是否从协议中提款的 bool 变量 isWithdraw。

 ```
 struct Order{
         address pledger; 
         address collection;
         uint256 tokenId; 
         uint256 nftType;
         bool isWithdraw;
 }
 function pledge(address _collection, uint256 _tokenId, uint256 _nftType) external nonReentrant{
         pledgeInternal(_collection, _tokenId, _nftType);
 }
 function pledgeInternal(address _collection, uint256 _tokenId, uint256 _nftType) internal whenNotPaused(1) returns(uint256){
         require(_nftType == 721 || _nftType == 1155, "don't support this nft type");
         if(_collection != address(punks)){
             transferNftInternal(msg.sender, address(this), _collection, _tokenId, _nftType);
         }else{
             _depositPunk(_tokenId);
             _collection = address(wrappedPunks);
         }
         require(collectionWhiteList[_collection].isCollectionWhiteList, "collection not insist");
 
         counter = counter.add(1);
         uint256 _orderId = counter;
         Order storage _order = allOrders[_orderId];
         _order.collection = _collection;
         _order.tokenId = _tokenId;
         _order.nftType = _nftType;
         _order.pledger = msg.sender;
 
         ordersMap[msg.sender].push(counter);
 
         emit Pledge(_collection, _tokenId, _orderId, msg.sender);
         return _orderId;
 }
 ```

用户可以调用 XNFT.withdrawNFT 将抵押物取回，不过需要用户这一 Order 中没有未还清的贷款。取出 NFT 后，会将 Order.isWithdraw 字段设置为 true，用于标记这一 order 的抵押物已经被取出

```
    function withdrawNFT(uint256 orderId) external nonReentrant whenNotPaused(2){
        LiquidatedOrder storage liquidatedOrder = allLiquidatedOrder[orderId];
        Order storage _order = allOrders[orderId];
        if(isOrderLiquidated(orderId)){
            ...
        }else{
            require(!_order.isWithdraw, "the order has been drawn");
            require(_order.pledger != address(0) && msg.sender == _order.pledger, "withdraw auth failed");
            uint256 borrowBalance = controller.getOrderBorrowBalanceCurrent(orderId);
            require(borrowBalance == 0, "order has debt");
            transferNftInternal(address(this), _order.pledger, _order.collection, _order.tokenId, _order.nftType);
        }
        _order.isWithdraw = true;
        emit WithDraw(_order.collection, _order.tokenId, orderId, _order.pledger, msg.sender);
    }
```

抵押在 XNFT 合约中的 NFT 可以作为抵押物在 XToken 合约进行贷款，目前只支持 ETH 作为交付手段。进行抵押借贷时会对用户的抵押物进行价格计算以及 TLV 等账户信息状态进行更新检查。

漏洞点就出现在贷款接口 XToken.borrow 函数中，在前期的贷款合法性检查前，没有检查 order.isWithdraw 这一字段，导致攻击者可以将已经取出的 NFT 重复作为抵押物进行贷款操作。

```
    function borrow(uint256 orderId, address payable borrower, uint256 borrowAmount) external{
        require(msg.sender == borrower || tx.origin == borrower, "borrower is wrong");
        accrueInterest();
        borrowInternal(orderId, borrower, borrowAmount);
    }
    function borrowInternal(uint256 orderId, address payable borrower, uint256 borrowAmount) internal nonReentrant{
        
        controller.borrowAllowed(address(this), orderId, borrower, borrowAmount);

        require(accrualBlockNumber == getBlockNumber(),"block number check fails");
        
        require(getCashPrior() >= borrowAmount, "insufficient balance of underlying asset");

        BorrowLocalVars memory vars;

        vars.orderBorrows = borrowBalanceStoredInternal(orderId);
        vars.orderBorrowsNew = addExp(vars.orderBorrows, borrowAmount);
        vars.totalBorrowsNew = addExp(totalBorrows, borrowAmount);
        
        doTransferOut(borrower, borrowAmount);

        orderBorrows[orderId].principal = vars.orderBorrowsNew;
        orderBorrows[orderId].interestIndex = borrowIndex;

        totalBorrows = vars.totalBorrowsNew;

        controller.borrowVerify(orderId, address(this), borrower);

        emit Borrow(orderId, borrower, borrowAmount, vars.orderBorrowsNew, vars.totalBorrowsNew);
    }
        function borrowAllowed(address xToken, uint256 orderId, address borrower, uint256 borrowAmount) external whenNotPaused(xToken, 3){
        require(poolStates[xToken].isListed, "token not listed");

        orderAllowed(orderId, borrower);

        (address _collection , , ) = xNFT.getOrderDetail(orderId);

        CollateralState storage _collateralState = collateralStates[_collection];
        require(_collateralState.isListed, "collection not exist");
        require(_collateralState.supportPools[xToken] || _collateralState.isSupportAllPools, "collection don't support this pool");

        address _lastXToken = orderDebtStates[orderId];
        require(_lastXToken == address(0) || _lastXToken == xToken, "only support borrowing of one xToken");

        (uint256 _price, bool valid) = oracle.getPrice(_collection, IXToken(xToken).underlying());
        require(_price > 0 && valid, "price is not valid");

        // Borrow cap of 0 corresponds to unlimited borrowing
        if (poolStates[xToken].borrowCap != 0) {
            require(IXToken(xToken).totalBorrows().add(borrowAmount) < poolStates[xToken].borrowCap, "pool borrow cap reached");
        }

        uint256 _maxBorrow = mulScalarTruncate(_price, _collateralState.collateralFactor);
        uint256 _mayBorrowed = borrowAmount;
        if (_lastXToken != address(0)){
            _mayBorrowed = IXToken(_lastXToken).borrowBalanceStored(orderId).add(borrowAmount);  
        }
        require(_mayBorrowed <= _maxBorrow, "borrow amount exceed");

        if (_lastXToken == address(0)){
            orderDebtStates[orderId] = xToken;
        }
    }
```

### 攻击流程

由于 Xcarnival 的借贷信息存储在 order 结构体以及以 orderId 为索引的字典中，漏洞点出在缺少对 order 结构体中字段的检查，而每一次的借贷行为都会导致这一 order 不能继续借贷出更多的 ETH，所以攻击中重复了多次抵押、取款、借款的操作。

以其中两次关联的攻击为例：

抵押&取款：0x61a6a8936afab47a3f2750e1ea40ac63430a01dd4f53a933e1c25e737dd32b2f

贷款：0x51cbfd46f21afb44da4fa971f220bd28a14530e1d5da5009cfbdfee012e57e35

攻击者首先部署了一个自定义的 Xtoken，将其作为 pledgeAndBorrow 函数的参数传入（这一步可以替代）。

而后生成攻击合约，进行抵押和取款操作。

在第二个交易中，进行贷款操作完成攻击。

攻击者将上述步骤重复了多次，共获利3087个 ETH，约380万美元。

### 参考链接

https://twitter.com/peckshield/status/1541047171453034501

https://mp.weixin.qq.com/s/WEzNmTDGI2GUNrF7DJuZ6A

https://mp.weixin.qq.com/s/F2hpBNRzhZmfCn7BQanGOA