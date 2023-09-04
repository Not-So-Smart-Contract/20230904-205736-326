>**了解智能合约审计和最佳实践请**[点击查看文档](https://safful.com/) 

# UniswapX 源码分析

# 设计原理

UniswapX 旨在通过将路由复杂性外包给第三方填充者的开放网络来解决，然后第三方填充者竞争使用 AMM 矿池或自己的私人库存等链上流动性来填充掉期。
借助 UniswapX，交换者将能够使用 Uniswap 界面，而不必担心自己是否获得最佳价格，并且交易将始终在链上透明地记录和结算。所有订单均由 Uniswap 智能订单路由器支持，这迫使填充者与 Uniswap v1、v2、v3 以及一旦启动后的 v4 竞争。

# 优势

- 通过聚合流动性来源获得更好的价格
- 无 gas 交换
- 防止 MEV（最大可提取值）
- 交易失败无需支付任何费用
- 在接下来的几个月中，UniswapX 将扩展到无 Gas 跨链交换。

# 工作原理

首先，假设 Alice（交换者）想要将 1 ETH 交换为 USDC。Alice 向（潜在的填充者）Bob、Charlie 和 Danielle 请求报价：

- Bob 提出以 1,000 USDC 购买 Alice 的 ETH
- Charlie 现有 999 USDC
- Danielle 现有 998 USDC
- Alice 还可以直接通过 Uniswap v3 将她的 1 ETH 兑换成 997 USDC

Alice 接受 Bob 的 1,000 USDC 报价，并签署订单。
该订单包括最大值（由 Bob 的报价 1,000 USDC 设置）和最小值 997 USDC（由 Uniswap 智能订单路由器 API 设置）。
Bob 可以使用他自己的 USDC 或将 Alice 的 1 个 ETH 路由到各种链上流动性场所（Uniswap 协议、Sushiswap 等）来填写 Alice 的订单。
Bob 决定使用自己的 USDC 来满足 Alice 的订单，并向 Alice 发送 1,000 USDC 以换取她的 1 ETH。
如果 Bob 决定放弃他的提议，Alice 不需要提交新的订单和签名。
相反，她现有的订单会自动更新，向任何能给她 999 USDC 作为回报的人提供 1 ETH。
一个区块已经过去，现在 Charlie 和 Danielle（以及参与 UniswapX 系统的任何其他填充者）都不愿意以 999 USDC 的价格填写 Alice 的订单。另一个以太坊区块（12 秒）到期后，Alice 的 1 ETH 可兑换 998 USDC。
突然，Danielle 意识到，通过将 Alice 的交易发送到 Uniswap v3 和 Sushiswap 的组合，她可以以 998 USDC 的价格填写 Alice 的 1 ETH 卖单，同时仍然为自己赚取 1 USDC 的利润。
Danielle 代表 Alice 将 Alice 的 1 ETH 发送到 Uniswap v3 和 Sushiswap，将 998 USDC 返还给 Alice，并为自己保留剩余的 1 USDC 输出。

# 交易流程

UniswapX 是一个去中心化交易协议，利用 Permit2 代币授权合约引入了基于签名的授权和转账功能，适用于任何 ERC20 代币。此外，UniswapX 还使用 Reactor 合约进行链上结算。Reactor 合约负责验证交易是否符合用户指定的参数，并可以撤销不符合条件的交易。要参与 UniswapX 的交易，兑换者首先必须授权 Permit2 合约。
兑换者无需手动创建和提交交易，而是对交易订单签名，指定以下参数：

1. 输入代币（支付代币）
2. 输出代币（获取代币）
3. 输入（输出）数量
4. 初始输出（输入）金额
5. 最低输出（输入）数量
6. 衰减函数
7. 兑换期限
8. 授权 UniswapX Reactor 合约代表其使用代币

这些订单由 MEV 搜索者、做市商和 / 或其他链上代理（统称为填单者）接收，并将其发送到 Reactor 合约。通过在链上提交兑换者的订单，填单者代表兑换者支付 Gas 费用。这些费用会反映在执行价格中，以补偿 Gas 成本。
Reactor 合约调用填单者的 Executor 合约，其中包含特定的订单执行逻辑。一旦确定资产来源，Executor 合约将资产发送到兑换者的地址，并从兑换者地址提取资金。最后，Reactor 合约验证订单是否满足条件。
UniswapX 没有规定填单者如何填充兑换者的订单。流动性可以来自 Uniswap 或其他去中心化交易所的链上流动性池、链下流动性源或其他 UniswapX 订单。多个订单可以捆绑到同一笔交易中，并且其他操作可以在链上原子执行。

# 关键源码解析

![](https://cdn.nlark.com/yuque/0/2023/png/97322/1693192482017-9747d764-02a4-40c2-8da6-9abf14f20fec.png#averageHue=%23252020&clientId=ue1aa3e09-9857-4&from=paste&id=u6ae768c1&originHeight=720&originWidth=1101&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=uea7664a9-68de-4d4b-8106-dd0fde1005b&title=)

由于填充者需要代替交换者提交 gas，所以可以通过批量执行订单的方式来减少一次交易带来的手续费损耗。

![](https://cdn.nlark.com/yuque/0/2023/png/97322/1693192482843-4f24d00e-5954-46a5-9b65-c92aa1297604.png#averageHue=%23302221&clientId=ue1aa3e09-9857-4&from=paste&id=u29b97a54&originHeight=664&originWidth=1280&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=ueac0b5c3-f883-482f-9296-a4548750037&title=)

\_fill 函数中处理具体订单的执行逻辑，这里存在两种情况，如果填充者使用自己个人持仓来完成用户的兑换，则不需要使用回调合约，直接进行资金对换；否则需要在回调合约中来处理具体逻辑，例如到其他交易池中进行兑换等。

![](https://cdn.nlark.com/yuque/0/2023/png/97322/1693192482168-a983dc8c-161e-460c-9d7d-384c63ac4758.png#averageHue=%232f2221&clientId=ue1aa3e09-9857-4&from=paste&id=u2130fe13&originHeight=401&originWidth=1280&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u2dcd07aa-22ee-4a10-8dfa-890b2f9dcf6&title=)

合约使用 validate 函数来验证填充者是否是订单的指定填充者。

![](https://cdn.nlark.com/yuque/0/2023/png/97322/1693192482392-e738bb3d-ff0b-43d4-af17-cd1011d95682.png#averageHue=%23272120&clientId=ue1aa3e09-9857-4&from=paste&id=u638cd6d6&originHeight=360&originWidth=1280&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u9771623f-505e-42ca-88c0-0f8c9abe3ea&title=)
![](https://cdn.nlark.com/yuque/0/2023/png/97322/1693192482342-b1bc4b8e-21e4-4a19-bb10-8dafb36d3a7e.png#averageHue=%232c2221&clientId=ue1aa3e09-9857-4&from=paste&id=uae991c06&originHeight=720&originWidth=1152&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u5cf0ef1a-7074-495a-8ec1-269d1c60049&title=)

合约使用了 permit2 库来完成签名的校验和代币的转账，以此保证交换者的钱不会被随意转走。

![](https://cdn.nlark.com/yuque/0/2023/png/97322/1693192483164-eadc923f-ce1b-4aff-97ce-1703780a8497.png#averageHue=%23282120&clientId=ue1aa3e09-9857-4&from=paste&id=u070a1e80&originHeight=510&originWidth=1280&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=ube9ad8ce-3546-46a0-a445-6553d1dbf01&title=)

若填充者选择使用个人持仓完成订单，则会直接将代币从填充者地址转移到交换者地址。

![](https://cdn.nlark.com/yuque/0/2023/png/97322/1693192483126-da836998-ad61-4a3c-b868-c1f5e3892b49.png#averageHue=%23242321&clientId=ue1aa3e09-9857-4&from=paste&id=u46aa7771&originHeight=412&originWidth=1280&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u77e97734-99cb-43f5-8942-d15ef769b46&title=)

在回调合约的回调函数执行完成后，调用 check 函数校验用户是否收到了足够的代币，若不满足足够的代币，则交易整个回退。
总结，合约中涉及到的只有关于链上的逻辑，由于用户并不需要支付 gas 费来完成这一笔交易，所以前期的多数操作选择在链下进行，包括用户的交换请求发送和对交易进行签名等。uniswapX 选择在链下将用户的交换请求发送给填充者，而一旦填充者接受了填充请求，则由填充者将交易发送到链上，并从中赚取差值作为利润。

![](https://cdn.nlark.com/yuque/0/2023/png/97322/1693192483335-05711bef-257f-42ee-8477-36ce660e670d.png#averageHue=%23f8f8f8&clientId=ue1aa3e09-9857-4&from=paste&id=ud3cbfb3e&originHeight=720&originWidth=990&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u48adbdca-81f6-4979-aec4-17f791888ea&title=)
