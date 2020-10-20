# CoFiX兑换协议产品文档

## 1、简介
本文档是对“CoFiX：一种可计算的金融交易模型” 白皮书所描述的金融交易模型的实现设计。主要包括价格计算、交易、做市商、异常处理等机制和前端页面交互定义。

## 2、系统角色

角色 | 定义
---|---
交易者 | 参与 CoFiX 交易的对象，可以是某个以太坊钱包或者合约地址
做市商 | 参与 CoFiX 做市的对象，是交易者的对手盘，为交易提供流动性，可以是某个以太坊钱包地址或者合约地址
治理者 | 参与 CoFiX系统治理的对象，是 CoFiX的治理代币持有者，通过发起投票修改和升级 CoFiX系统。（前期由管理员多签账户代理）

## 3、价格计算机制  

### 3.1、价格来源

CoFiX 协议的价格来源于 NEST 预言机，每个交易池对应一个 NEST 预言机价格源，例如：

ETH: USDT 交易池，价格从 NEST 的 ETH/USDT 预言机调用。

ETH：HBTC 交易池，价格从 NEST 的 ETH/HBTC 预言机调用。

在NEST系统里，一个报价对对应一个预言机，详见NEST预言机文档。

### 3.2、价格补偿系数K

由于 NEST 价格和均衡价格存在一定的偏差和延时(可以理解成去中心化的成本)，因 此 CoFiX 在应用该价格时需要对风险给出一些补偿，确保做市商有动力持续做市。补偿系数记为 K，该系数与波动率σ和延时变量 T 有关，当交易者进行交易时，并不是直接使用NEST价格 P，而是使用 ：

买入时：![](http://latex.codecogs.com/svg.latex?P_{b}^{'}=P*(1+K))

或者

卖出时：![](http://latex.codecogs.com/svg.latex?P_{s}^{'}=P*(1-K))

这里的买入和卖出都是相对价格P所对应的ETH对手资产来说的，例如价格P是以ETH对手资产为本位的， 即一单位ETH等于多少USDT、HBTC等资产，则买入和卖出指的是买入和卖出ETH（卖出时其实就是买入ETH的对手资产）。

同样做市商在进入和退出做市时，计算净值用 的价格变量也是![](http://latex.codecogs.com/svg.latex?{P}\')而不是 P，这一价格称之为交易价格。详细计算方式见附录“CoFiX价格偏差系数（K）计算说明”

### 3.3、预估价格和执行价格

由于以太坊一笔交易由发起到执行需要一定的等待时间，并且 NEST 预言机价格一直处于更新状态，因此交易发起和实际交易打包成功时的价格会存在一定的差异。这里我们先定义一下预估价格和执行价格。


 价格 | 说明
---|---
预估价格：![](http://latex.codecogs.com/svg.latex?P_{es})| 用户或者做市商在发起交易时看到的参照价格，取自 NEST 预言机最新历史价格。这个价格主要是前端用来进行交易价格展示以及兑换金额、认购份额、赎回资产数量的预估计算。
执行价格：![](http://latex.codecogs.com/svg.latex?P_{ex})| 交易执行时，实际调用的 NEST 预言机最新价格。

由预估价格和执行价格可以计算实际成交价差![](http://latex.codecogs.com/svg.latex?P_{d})：

![](http://latex.codecogs.com/svg.latex?P_{d}=\left|P_{ex}-P_{es}&space;\right|/P_{es})

当前设定当![](http://latex.codecogs.com/svg.latex?P_{d}>1%)时发出的交易会被revert

### 3.4、预言机价格调用费

当向 CoFiX 发起交易、赎回、认购操作时，会向 NEST Protocol 调用预言机价格、因此用户需要向预言机支付对应的调用费，收费标准取决于 NEST Protocol，目前调价格的收费是 0.01 ETH。

## 4、做市商机制

做市商是 CoFiX交易流动性提供者，可以通过转入资产为某个交易池提供流动性来获得收益。CoFiX 支持单边资产做市，通过对应交易池的基金净值以及份额来进行资产管理。

### 4.1、做市商份额

份额代表着 CoFiX某个资产池所拥有的做市资产比例凭证。当做市商向交易池转入资产进行做市时，获得 XToken。当做市商想退出时，通过转出并销毁 XToken，赎回对应价值的资产。

每个做市资金池拥有独立的 XToken，按照交易对创建时间顺序对 XToken 进行命名为 ![](http://latex.codecogs.com/svg.latex?XT_{1})、![](http://latex.codecogs.com/svg.latex?XT_{2})、![](http://latex.codecogs.com/svg.latex?XT_{3}) …

### 4.2、交易池创建和初始化

任何人都可以选择一个 NEST 预言机来创建对应的 CoFiX 交易池，该交易池使用对应预言机的价格，并且在创建时进行净值和份额的初始化。以  ETH:USDT 资产池为例，当市商池在创建时：

净值初始化为 1

初始发行份额数量![](http://latex.codecogs.com/svg.latex?S_{0})为：

![](http://latex.codecogs.com/svg.latex?S_{0}=A_{u}/P_{b}^{'}&plus;A_{e})

其中：![](http://latex.codecogs.com/svg.latex?P_{b}^{'}=P*(1+K))，P 为ETH/USDT的当前价格 ，K为当前补偿系数

![](http://latex.codecogs.com/svg.latex?A_{e})为做市资产 ETH初始数量 

![](http://latex.codecogs.com/svg.latex?A_{u})为做市资产 USDT 初始数量 

### 4.3、做市商净值

净值代表着某个资产池中，每个份额（XToken）的 ETH 本位价值。每当有交易、认购、赎回操作时，对净值进行更新。以  ETH:USDT 资产池为例，净值计算公式为：

认购时：![](http://latex.codecogs.com/svg.latex?N_{p}=(A_{u}/P_{s}^{'}&plus;A_{e})/S)

赎回时：![](http://latex.codecogs.com/svg.latex?N_{p}^{'}=(A_{u}/P_{b}^{'}&plus;A_{e})/S)

其中：

S为ETH:USDT 资产池总发行份额 

ETH 做市资产总量 = ![](http://latex.codecogs.com/svg.latex?A_{e})

USDT 做市资产总量 = ![](http://latex.codecogs.com/svg.latex?A_{u})

![](http://latex.codecogs.com/svg.latex?P_{b}^{'}=P*(1+K))

![](http://latex.codecogs.com/svg.latex?P_{s}^{'}=P*(1-K))

P 为ETH/USDT的当前价格 ，K为当前补偿系数

### 4.4、份额认购

认购代表着参与某个交易池做市，在转入资产同时获得份额对应 XToken。以 ETH: USDT 资产池为例：

Alice 转入 a 个ETH 参与认购做市，那么获得的 XToken 数量![](http://latex.codecogs.com/svg.latex?S_{1})为：

![](http://latex.codecogs.com/svg.latex?S_{1}=a/N_{p})

其中：![](http://latex.codecogs.com/svg.latex?N_{p})为当前认购时候的净值 

同理 Alice 若转入 b 个 USDT 参与做市，那么此时 alice 获得的 XToken 数量![](http://latex.codecogs.com/svg.latex?S_{2})为：

![](http://latex.codecogs.com/svg.latex?S_{2}=b/P_{b}^{'}/N_{p})

其中：![](http://latex.codecogs.com/svg.latex?N_{p})为当前认购时候的净值，![](http://latex.codecogs.com/svg.latex?P_{b}^{'}=P*(1+K))

### 4.5、份额赎回

赎回代表着退出某个交易池做市，通过转入对应 XToken 按照当前净值兑换任意一边资产，转入的 XToken 会进行销毁。以 ETH: USDT 资产池为例：

Alice 转入 c 个 XToken ，那么可以兑换出来的 ETH 数量 e 为：

![](http://latex.codecogs.com/svg.latex?e=c*N_{p}^{'}*(1-\theta&space;))

其中：![](http://latex.codecogs.com/svg.latex?N_{p}^{'})为赎回时的净值，![](http://latex.codecogs.com/svg.latex?\theta)为交易手续费系数。

可以兑换出来的 USDT 数量 u 为：

![](http://latex.codecogs.com/svg.latex?u=c*N_{p}^{'}*P_{s}^{'}*(1-\theta&space;))

其中![](http://latex.codecogs.com/svg.latex?N_{p}^{'})为赎回时的净值，![](http://latex.codecogs.com/svg.latex?\theta)为交易手续费系数。

![](http://latex.codecogs.com/svg.latex?P_{s}^{'}=P*(1-K))，P 为ETH/USDT的当前价格 ，K为当前补偿系数

## 5、交易机制

### 5.1、ETH对ERC-20 交易

以 Alice 使用 ETH 兑换 USDT 以及 使用 USDT 兑换 ETH 为例：

交易者Alice 使用 a 个 USDT 兑换出 x 个 ETH，计算过程为：

![](http://latex.codecogs.com/svg.latex?x=(a/P_{b}^{'})*(1-\theta))

其中：![](http://latex.codecogs.com/svg.latex?P_{b}^{'}=P*(1+K))，P为ETH/USDT的当前价格 ；K为当前补偿系数；![](http://latex.codecogs.com/svg.latex?\theta)为交易手续费系数。

交易者Alice 使用 b 个 ETH 兑换出 y 个 USDT，计算过程为：

![](http://latex.codecogs.com/svg.latex?y=b*P_{s}^{'}*(1-\theta))

其中：![](http://latex.codecogs.com/svg.latex?P_{s}^{'}=P*(1-K))，P为ETH/USDT的当前价格 ；K为当前补偿系数；![](http://latex.codecogs.com/svg.latex?\theta)为交易手续费系数。

### 5.2、ERC-20 对 ERC-20交易

当有多个不同市商池时，可以通过调用2个 ETH:ERC-20 市商池，通过一笔交易完成 ERC-20:ERC-20 兑换。

原理示例：

![](https://nestapp.oss-cn-beijing.aliyuncs.com/NestCoreForGitHub/cofixImage.png)

计算过程以 Alice 使用 USDT 兑换 HBTC 过程为例：

那么, 当Alice 使用 a 个 USDT兑换出 y 个 HBTC, 计算过程为：

1. 使用 USDT 兑换出 x 个 ETH

![](http://latex.codecogs.com/svg.latex?x=(a/P_{1,b}^{'})*(1-\theta))

其中：![](http://latex.codecogs.com/svg.latex?P_{1,b}^{'}=P_{1}*(1&plus;K_{1}))，![](http://latex.codecogs.com/svg.latex?P_{1})为ETH/USDT 预言机价格, ![](http://latex.codecogs.com/svg.latex?K_{1})为ETH:USDT 交易池价格补偿系数。

2. 使用 x 个  ETH 兑换出 y 个 HBTC

![](http://latex.codecogs.com/svg.latex?y=x*P_{2,s}^{'}*(1-\theta&space;))

其中：![](http://latex.codecogs.com/svg.latex?P_{2,s}^{'}=P_{2}*(1-K_{2})),![](http://latex.codecogs.com/svg.latex?P_{2})为ETH/HBTC 预言机价格,![](http://latex.codecogs.com/svg.latex?K_{2})为ETH:HBTC交易池价格补偿系数; ![](http://latex.codecogs.com/svg.latex?\theta)为交易手续费系数。

3. 1 和 2步骤合并可以得出，使用 a 个 USDT 可以兑换出的 HBTC数量为

![](http://latex.codecogs.com/svg.latex?y=(a/P_{1,b}^{'})*P_{2,s}^{'}*(1-\theta&space;)^{2})

其中：![](http://latex.codecogs.com/svg.latex?P_{1,b}^{'}=P_{1}*(1&plus;K_{1}))，![](http://latex.codecogs.com/svg.latex?P_{1})为ETH/USDT预言机价格, ![](http://latex.codecogs.com/svg.latex?K_{1})为ETH:USDT交易池价格补偿系数;![](http://latex.codecogs.com/svg.latex?P_{2,s}^{'}=P_{2}*(1-K_{2}))，![](http://latex.codecogs.com/svg.latex?P_{2})为ETH/HBTC预言机价格 ,![](http://latex.codecogs.com/svg.latex?K_{2})为ETH:HBTC交易池价格补偿系数; ![](http://latex.codecogs.com/svg.latex?\theta)为交易手续费系数。

## 6、风险控制

### 6.1、交易延时控制

考虑到以太坊存在交易拥堵，长时间无法打包的情况。因此，当一笔交易发出去后，实际成交的即时价格与交易发起时的预估价格可能会存在较大的偏差，因此引入一个变量，交易有效时间![](http://latex.codecogs.com/svg.latex?t)。

1. 交易发起时，时间为![](http://latex.codecogs.com/svg.latex?t_{0})
2. 交易打包成功时，时间为![](http://latex.codecogs.com/svg.latex?t_{1})
3. 那么，当 ![](http://latex.codecogs.com/svg.latex?t_{1}-t_{0}>t) 时，此笔交易将视为交易不成功，返回用户交易资产。
4. 当前设定![](http://latex.codecogs.com/svg.latex?t=600s)

### 6.2、停机机制

当波动率上升到一个极端情况，或者 NEST 系统被攻击时，需要 CoFiX 启动应急方案，主要是为了保护交易双方，特别是做市商。此时系统可以设置停机机制，即触发 一类条件时暂停交易。目前设置的停机条件有5类，当达到停机条件时，发出的交易会被revert: 

1. ![](http://latex.codecogs.com/svg.latex?K_{0})值超过一个范围，目前设定为![](http://latex.codecogs.com/svg.latex?K_{0}>5%)；
2. 秒级波动率![](http://latex.codecogs.com/svg.latex?\sigma&space;)上升到一个范围，目前设定为![](http://latex.codecogs.com/svg.latex?\sigma>0.1%)；
3. 预言机价格间隔：![](http://latex.codecogs.com/svg.latex?T>900s)；
4. 交易打包延迟：![](http://latex.codecogs.com/svg.latex?t>600s)；
5. 实际成交价差：![](http://latex.codecogs.com/svg.latex?P_{d}>1%)。

具体停机参数的计算见附录”CoFiX价格偏差系数（K）计算说明”

## 7、流动性挖矿激励系统

### 7.1、矿池总量:

整个系统总量无上限，但通胀率会随着时间推移逐渐下降。

系统有三个矿池，即交易矿池A，做市商矿池B，节点矿池C，其数量如下：

矿池A：动态出矿，总量不固定，单笔交易的出矿量和佣金、最近300个区块的交易密度、做市商资金池的规模及平衡性相关，每笔交易出矿中，80%由交易者获得，10%分给对应的做市商矿池，10%归属节点矿池。

矿池B：由固定出矿和浮动出矿两部分，固定出矿部分，每个区块出矿9个，每240万个区块后衰减到上期值的80%，960万个区块后不再衰减，按照每个块出矿3.6864；浮动部分为交易出矿的10%。

矿池C：当由固定出矿和浮动出矿两部分，固定出矿部分，每个区块出矿1个，每240万个区块后衰减到上期值的80%，960万个区块后不再衰减，按照每个块出矿0.4096；浮动部分为交易出矿的10%。

### 7.2、交易者挖矿模型：

交易者每笔挖矿都会产出一定的CoFi，其中80%归该交易者，10%归节点矿池，10%归做市商矿池。每笔产出的CoFi主要取决于该笔交易支付的佣金：

1. 假设该笔交易的规模为![](http://latex.codecogs.com/svg.latex?X_{t})(ETH)，佣金为![](http://latex.codecogs.com/svg.latex?Y_{t}=X_{t}*\theta&space;)(ETH)；
2. 单位佣金（1ETH）挖出的CoFi标准量为at  , at的计算公式如下：

![](http://latex.codecogs.com/svg.latex?a_{t}=\frac{b_{t}*\varphi*2400000}{X_{t}*N_{p}*r})

公式注释：

(1) ![](http://latex.codecogs.com/svg.latex?b_{t})为做市商矿池在当前时刻的单位区块出矿量；

(2) ![](http://latex.codecogs.com/svg.latex?X_{t})为当前交易对池子的总份额；

(3) ![](http://latex.codecogs.com/svg.latex?N_{p})为当前交易对池子的净值；

(4) ![](http://latex.codecogs.com/svg.latex?\varphi)为对应交易对的出矿系数，也即当前交易对池子的出矿占比。![](http://latex.codecogs.com/svg.latex?\varphi_{1})为ETH-USDT交易对的出矿系数，![](http://latex.codecogs.com/svg.latex?\varphi_{2})为ETH-HBTC交易对的出矿系数，当前

![](http://latex.codecogs.com/svg.latex?\varphi&space;_{1}=2/3\approx0.67,\varphi&space;_{2}=1/3\approx&space;0.33)

(5) ![](http://latex.codecogs.com/svg.latex?r)为做市资产的预期收益率, 当前设![](http://latex.codecogs.com/svg.latex?r=0.3)。

3. 考虑到连续若干笔交易规模过大，会导致出矿不受控制，因此我们设计了密度衰减指标，其核心参数如下：

(1) 单笔交易触发密度衰减的阈值为![](http://latex.codecogs.com/svg.latex?L*\theta*a_{t})，其中![](http://latex.codecogs.com/svg.latex?L=X_{t}*N_{p}*I)，I为衰减阈值系数，当前设定I=0.002，![](http://latex.codecogs.com/svg.latex?L)的最小值为100（ETH）；

(2) 假设本次交易和上次交易之间的区块间隔为s，则密度参数:

![](http://latex.codecogs.com/svg.latex?f_%7Bt%7D=%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D%20f_%7Bt-1%7D*(300-s)/300&plus;y_%7Bt%7D*a_%7Bt%7D&s%5Cleq%20300%5C%5C%20y_%7Bt%7D*a_%7Bt%7D&s%3E%20300%5C%5C%5Cend%7Bmatrix%7D%5Cright.)

其中：![](http://latex.codecogs.com/svg.latex?f_{0}=0)，本次交易和上次交易在同一区块或本次交易是第一笔交易时![](http://latex.codecogs.com/svg.latex?s=0)；

(3) 为了确保交易池两端资产的平衡，在挖矿中加入一个平衡系数![](http://latex.codecogs.com/svg.latex?\lambda&space;)，![](http://latex.codecogs.com/svg.latex?\lambda&space;)计算如下：

假设交易者用资产![](http://latex.codecogs.com/svg.latex?V_{x})兑换资产![](http://latex.codecogs.com/svg.latex?V_{y})， 交易池内资产![](http://latex.codecogs.com/svg.latex?V_{x})总量为![](http://latex.codecogs.com/svg.latex?U_{x})，资产![](http://latex.codecogs.com/svg.latex?V_{y})总量为![](http://latex.codecogs.com/svg.latex?U_{y})（按交易时的价格换算成ETH），![](http://latex.codecogs.com/svg.latex?\lambda&space;)的值的大小取决于![](http://latex.codecogs.com/svg.latex?U_{x}/U_{y})的值的大小，具体公式如下： 

![](http://latex.codecogs.com/svg.latex?%5Clambda=%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D0.25%20&%20U_%7Bx%7D/U_%7By%7D%5Cgeqslant%2010%20%5C%5C%200.50%20&%2010%3E%20U_%7Bx%7D/U_%7By%7D%5Cgeqslant%203%20%5C%5C1.00%20&%203%3E%20U_%7Bx%7D/U_%7By%7D%5Cgeqslant%200.33%20%20%5C%5C2.00%20&%200.33%3E%20U_%7Bx%7D/U_%7By%7D%5Cgeqslant%200.1%20%20%5C%5C4.00%20&%20U_%7Bx%7D/U_%7By%7D%3C%200.1%20%20%5C%5C%5Cend%7Bmatrix%7D%5Cright.)

(4) 出矿量公式：

![](http://latex.codecogs.com/svg.latex?y_{t})对应的交易者出矿量![](http://latex.codecogs.com/svg.latex?A(y_{t}))计算如下：

![](http://latex.codecogs.com/svg.latex?A(y_%7Bt%7D)=%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D0.8*y_%7Bt%7D*a_%7Bt%7D*%5Clambda&f_%7Bt%7D%5Cleq%20L*%5Ctheta*a_%7Bt%7D%5C%5C0.8*y_%7Bt%7D*a_%7Bt%7D*L*%5Ctheta*a_%7Bt%7D*(2f_%7Bt%7D-L*%5Ctheta*a_%7Bt%7D)/%7Bf_%7Bt%7D%7D%5E%7B2%7D*%5Clambda&f_%7Bt%7D%3EL*%5Ctheta*a_%7Bt%7D%5C%5C%5Cend%7Bmatrix%7D%5Cright.)

公式注释：

(1) 0.125![](http://latex.codecogs.com/svg.latex?A(y_{t}))流向节点矿池，记为![](http://latex.codecogs.com/svg.latex?R(y_{t}))，即出矿的10%；

(2) 另外0.125![](http://latex.codecogs.com/svg.latex?A(y_{t}))流向该笔交易对应的做市商矿池，记为![](http://latex.codecogs.com/svg.latex?I_{j}(y_{t})),   j是第j个交易池，参见做市商挖矿模型。

### 7.3、做市商挖矿模型：

假设有![](http://latex.codecogs.com/svg.latex?q)个交易池可以参与挖矿，对应的XToken依次记为 ![](http://latex.codecogs.com/svg.latex?XT_{1})、![](http://latex.codecogs.com/svg.latex?XT_{2})…![](http://latex.codecogs.com/svg.latex?XT_{n})，则做市商矿池等比例分成![](http://latex.codecogs.com/svg.latex?q)个子矿池，即![](http://latex.codecogs.com/svg.latex?B_{1})、![](http://latex.codecogs.com/svg.latex?B_{2})...![](http://latex.codecogs.com/svg.latex?B_{q})。某个池子![](http://latex.codecogs.com/svg.latex?B_{j})，以及做市商m在时刻![](http://latex.codecogs.com/svg.latex?h_{m,t-1})(![](http://latex.codecogs.com/svg.latex?h_{m,t-1})表示做市商m的第t-1次操作对应的时刻)存入![](http://latex.codecogs.com/svg.latex?B_{j})矿池合约的![](http://latex.codecogs.com/svg.latex?X_{j})的数量余额![](http://latex.codecogs.com/svg.latex?x_{m,t-1})，则做市商在其后一个时刻![](http://latex.codecogs.com/svg.latex?h_{m,t})，无论执行存入、取出、领取三种操作的任何一种，都可以挖出CoFi，且其出矿量为：

![](http://latex.codecogs.com/svg.latex?B_{m,t}(x_{m,t-1})=(G_{m,t}-G_{m,t-1})*x_{m,t-1})

公式注释：

1. ![](http://latex.codecogs.com/svg.latex?x_{m,t-1})本次操作前存入合约余额 ；
2. ![](http://latex.codecogs.com/svg.latex?G_{m,t})为做市商m本次操作的出矿系数，![](http://latex.codecogs.com/svg.latex?G_{m,t-1})为该做市商地址的上一次操作的出矿系数，![](http://latex.codecogs.com/svg.latex?G_{t})是一个公共变量，任何做市商在执行存入、取出、领取操作都可以改变这个参数，![](http://latex.codecogs.com/svg.latex?G_{t})计算方式如下：

![](http://latex.codecogs.com/svg.latex?G_{t}=G_{t-1}&plus;(b_{t}*\varphi*(h_{t}-h_{t-1})&plus;\Sigma&space;I_{j}(Y_{i}))/X_{t})

其中：![](http://latex.codecogs.com/svg.latex?G_{0}=0)，![](http://latex.codecogs.com/svg.latex?\Sigma&space;I_{j}(Y_{i}))的求和区间为![](http://latex.codecogs.com/svg.latex?h_{t})到![](http://latex.codecogs.com/svg.latex?h_{t-1}), 

公式注释：

(1) 为矿池B在![](http://latex.codecogs.com/svg.latex?h_{t})时刻的单位区块固定部分的出矿量,初始![](http://latex.codecogs.com/svg.latex?b_{0}=9)，过240万个区块衰减到上期值的80%，960万个区块后不再衰减，按照每个块出矿3.6864；

(2) ![](http://latex.codecogs.com/svg.latex?I_{j}(Y_{i}))为![](http://latex.codecogs.com/svg.latex?h_{t-1})到![](http://latex.codecogs.com/svg.latex?h_{t})期间j交易池交易挖矿分给做市商的部分；

(3) ![](http://latex.codecogs.com/svg.latex?X_{t})为t时刻j矿池存入的![](http://latex.codecogs.com/svg.latex?XT_{j})的总余额；

(4) ![](http://latex.codecogs.com/svg.latex?\varphi&space;)为对应矿池j的出矿系数，也即当前交易对池子的出矿占比。

### 7.4、节点挖矿模型：

节点总量为100个，与做市商类似，节点持有者将节点存在节点矿池合约，并在每次存入、取出、领取操作时根据给定的算法获得对应的CoFi。和做市商类似，节点矿池也存在一个出矿系数，每次操作基于 该挖矿系数出矿。

假设某节点持有人m在时刻![](http://latex.codecogs.com/svg.latex?h_{m,t-1})，(![](http://latex.codecogs.com/svg.latex?h_{m,t-1})表示m的第t-1次操作对应的时刻)，存入矿池合约的节点的数量余额![](http://latex.codecogs.com/svg.latex?n_{m,t-1})，则m在其后一个时刻![](http://latex.codecogs.com/svg.latex?h_{m,t})，无论执行存入、取出、领取三种操作的任何一种，都可以挖出CoFi，且其出矿量为：

![](http://latex.codecogs.com/svg.latex?C_{m,t}(n_{m,t-1})=(D_{m,t}-D_{m,t-1})*n_{m,t-1})

公式注释：

1. ![](http://latex.codecogs.com/svg.latex?n_{m,t-1})本次操作前存入合约余额；
2. ![](http://latex.codecogs.com/svg.latex?D_{m,t})为m本次操作的出矿系数，![](http://latex.codecogs.com/svg.latex?D_{m,t-1})为m上一次操作的出矿系数，![](http://latex.codecogs.com/svg.latex?D_{t})是一个公共变量，任何节点持有人在执行存入、取出、领取操作都可以改变这个参数，![](http://latex.codecogs.com/svg.latex?D_{t})计算方式如下：


![](http://latex.codecogs.com/svg.latex?D_{t}=D_{t-1}&plus;(c_{t}*(h_{t}-h_{t-1})&plus;\Sigma&space;R(y_{t}))/N_{t})

其中：![](http://latex.codecogs.com/svg.latex?D_{0}=0)

公式注释：

(1) ![](http://latex.codecogs.com/svg.latex?c_{t})为矿池C在![](http://latex.codecogs.com/svg.latex?h_{t})时刻的单位区块固定出矿部分，每个区块出矿1个，每240万个区块后衰减到上期值的80%，960万个区块后不再衰减，按照每个块出矿 0.4096 ；

(2) ![](http://latex.codecogs.com/svg.latex?\Sigma&space;R(y_{t}))为![](http://latex.codecogs.com/svg.latex?h_{t-1})到![](http://latex.codecogs.com/svg.latex?h_{t})期间交易池交易挖矿分给节点的部分；

(3) ![](http://latex.codecogs.com/svg.latex?N_{t})为t时刻存入节点矿池的节点token总余额。

### 7.5、CoFi分红及回购模型：

参与挖矿的交易池，其所有交易佣金ETH进入系统分红池，其中ɑ比例用于分红，![](http://latex.codecogs.com/svg.latex?1-\alpha&space;)用于回购，当前![](http://latex.codecogs.com/svg.latex?\alpha=20%)，回购机制在CoFi上线CoFiX后由DAO另行设计。

假设CoFi持有人m在时刻![](http://latex.codecogs.com/svg.latex?h_{m,t-1})，(![](http://latex.codecogs.com/svg.latex?h_{m,t-1})表示m的第t-1次操作对应的时刻)，存入分红合约的CoFi数量余额![](http://latex.codecogs.com/svg.latex?n_{m,t-1})，则m在其后一个时刻![](http://latex.codecogs.com/svg.latex?h_{m,t})，无论执行存入、取出、领取三种操作的任何一种，都可以分得ETH，且分红数量为：

![](http://latex.codecogs.com/svg.latex?E_{m,t}(n_{m,t-1})=(F_{m,t}-F_{m,t-1})*n_{m,t-1})

公式注释：

1. ![](http://latex.codecogs.com/svg.latex?n_{m,t-1})本次操作前存入合约余额；
2. ![](http://latex.codecogs.com/svg.latex?F_{m,t})为m本次操作的分红系数，![](http://latex.codecogs.com/svg.latex?F_{m,t-1})为m上一次操作的分红系数，![](http://latex.codecogs.com/svg.latex?F_{t})是一个公共变量，任何节点持有人在执行存入、取出、领取操作都可以改变这个参数，![](http://latex.codecogs.com/svg.latex?F_{t})计算方式如下：

![](http://latex.codecogs.com/svg.latex?F_{t}=F_{t-1}&plus;a*\Sigma&space;y/N_{t})

其中：![](http://latex.codecogs.com/svg.latex?F_{0}=0)

公式注释：

(1) ![](http://latex.codecogs.com/svg.latex?\Sigma&space;y)为![](http://latex.codecogs.com/svg.latex?h_{t-1})到![](http://latex.codecogs.com/svg.latex?h_{t})期间挖矿交易池中交易者支付的所有手续费；

(2) ![](http://latex.codecogs.com/svg.latex?N_{t})为t时刻存入分红合约的CoFi总余额。

## 8、治理  （待定）CoFiX DAO

### 8.1 、治理阶段

CoFiX 协议社区治理分为2个阶段。

第一阶段，由5个多签超级管理员地址来主导，负责对 CoFiX 早期阶段进行合约升级、参数调整。管理员地址主要由社区 KOL、早期投资人、合作项目方等多方组成。

第二阶段，超级管理员退出，进入社区治理阶段，合约升级、参数调整等决议需经过社区投票通过后方可执行。

### 8.2、 社区治理流程

1. 决议发布

- 设置好修改参数或者部署好升级相关的决议合约，并且开源。
- 质押一定数量的 CoFi Token，通过投票工厂合约对决议发起投票。
- 在 CoFiX 社区治理前端页面对投票进行发布。

2. 投票期

- 投票周期为 7 天。
- 投票周期内 CoFi Token 持有者可以直接使用存入在收益合约的 CoFi 进行投票。
- 投票率一旦超过 51% ，该合约进入公示期。

3. 公示期

- 投票公示期为 3天。
- 期间投票者可以对投票进行撤销。
- 撤销后，如果票选低于 51%，则退回到投票期。之后再次达到 51% 投票进入公示期 3 天重新倒计时。

4. 投票结束

- 投票持续 7 天没有达到 51% 投票率，则该决议被视为不通过。
- 决议通过 3 天公示期后，进入可激活状态，任何人可以激活使得改合约部署并生效。

### 8.3、 社区治理类型

投票类型 | 详细介绍
---|---
激活挖矿 | 激活某一个交易对挖矿功能
关闭挖矿 | 关闭某一个交易对挖矿功能
挖矿权重分配 | 重新分配各个交易对做市商挖矿比例
交易对手续费费率 | 调整处于挖矿模式下交易池的手续费
合约升级 | 对合约进行替换
DAO资产的管理 | DAO资产，回购、分红、流动性挖矿、出售、投资等等


# 附录

## CoFiX价格偏差系数（K）计算说明

### 一、说明

在进行闪兑兑换交易的时候，需要根据获取到的NEST Price来确定价格，由于NEST Price的生效事件和交易事件一般会有一个间隔，因此需要有一个偏差算法，偏差算法的公式里有一个偏差系数（K）作为变量,这里给出K值的影响因素和计算方法。

### 二、K值的影响因素

偏差系数根据最近一段时间的波动率，和当前时间与NEST Price生效时间的时间差来决定。

波动率（volatility）：根据最近连续50个区块的NEST价格来计算。

时间差（T）：当前所在区块和NEST Price生效区块的时间差。

### 三、环境影响

由于以太坊的出块速度可能发生变化，因此用区块作为时间并不能和实际时间形成对应，但是计算模型需要有一个恒定的参考，因此需要计算秒级波动率：

秒级波动率（sigma）：秒是现实世界相对确定的单位，因此计算模型采取秒计波动率作为输入单位。当前由于NEST Price只有区块价格，没有秒级价格数据， 只能由区块波动率和以太坊出块时间（timespan）来推算出秒级波动率。

以太坊出块时间（timespan）：表示以太坊的出块时间（秒），是一个可以配置的参数，根据目前的情况，timespan=14。

### 四、数据处理

1. NEST 预言机的价格是按照区块记录的，每个区块形成一个价格，由该区块内生效的报价按照一定的算法生成，该价格称之为区块价格或者 NEST-Price，如果该区块没有生效报价，则沿用上一个区块价格。这里计算波动率只取产生了有效价格区块的价格，以形成价格序列,同时保存每个区块对应的时间序列。
2. 剔除异常值，对于极为异常的价格数据，进行剔除。


### 五、计算背景

考虑到区块链计算的费用，我们采用指数加权移动平均模型（EWMA）来计算NEST有效报价的波动率。时间比较久远的变量值的影响力相对较低，时间比较近的变量值的影响力相对较高。EWMA需要保留上一个(t-1)的波动率值和最近连续两个价格数据即可，运行也相对简单很多，这是一个很好的减少内存和空间的做法。

### 六、K值的计算

假设NEST区块价格服从几何布朗运动模型或资产价格的对数收益服从几何布朗运动模型。设：

![](http://latex.codecogs.com/svg.latex?u_{t}=\frac{P_{t}}{P_{t-1}}-1)

则当期（t）波动率可指数加权移动平均模型（EWMA）来计算，公式为：

![](http://latex.codecogs.com/svg.latex?{\sigma&space;_{t}^{2}}=\lambda&space;{\sigma&space;_{t-1}^{2}}&plus;(1-\lambda)*{u_{t-1}^{2}}),![](http://latex.codecogs.com/svg.latex?t=2,3...)

其中:![](http://latex.codecogs.com/svg.latex?\lambda\in(0,1)), 因为NEST的有效报价有区块间隔，上述公式可调整为：

![](http://latex.codecogs.com/svg.latex?{\sigma&space;_{t}^{2}}=\lambda{\sigma_{t-1}^{2}}&plus;(1-\lambda)*\frac{{u_{t-1}^{2}}}{n_{t-1}*timespan}),![](http://latex.codecogs.com/svg.latex?t=2,3...)（1）

其中，![](http://latex.codecogs.com/svg.latex?n_{t})代表价格和价格之间的区块间隔数；timespan是以太坊出块平均时间；开始值![](http://latex.codecogs.com/svg.latex?\sigma^{2}&space;_{1}=\frac{u_{1}^{2}}{n_{1}*timespan})；当前设定![](http://latex.codecogs.com/svg.latex?\lambda=0.95&space;)。

以此权重计算，最新的50个波动率占90％以上的权重，各数值的影响力随时间呈指数式递减，时间越靠近当前时刻的数据影响力越大。同样EWMA模型可以对NEST有效价格数据进行处理，![](http://latex.codecogs.com/svg.latex?\bar{P}_{t}=\lambda\bar{P}_{t-1}&plus;(1-\lambda)P_{t})其中![](http://latex.codecogs.com/svg.latex?\bar{P}_{1}=P_{1})，![](http://latex.codecogs.com/svg.latex?t=2,3...)，根据NEST价格数据，价格![](http://latex.codecogs.com/svg.latex?P_{t})比EWMA价格![](http://latex.codecogs.com/svg.latex?\bar{P}_{t})高出或低于2.5％以上的概率仅为0.19％。因此可以用公式![](http://latex.codecogs.com/svg.latex?\left|P_{t}/\bar{P}_{t-1}-1&space;\right|<2.5%)来限制正常价格取值范围，排除异常价格数据。

我们再用线性公式来估计NEST价格的偏差的上限：

![](http://latex.codecogs.com/svg.latex?a(\sigma)=-0.0014687&plus;19.8898*\sigma&plus;gascost/10) （2）

其中：gascost=gas price*gasconsumed per transcation, 当前设定gascost=0.03，可根据实际情况进行调整。

T 为时间延迟：T=（打包成功区块高度-最近有效NEST价格所在区块高度）* timespan。

再根据公式（1）和（2）计算出![](http://latex.codecogs.com/svg.latex?\sigma)和![](http://latex.codecogs.com/svg.latex?a)，代入做市商预期亏损最大边界的计算公式：

![](http://latex.codecogs.com/svg.latex?K_{0}=\frac{a}{1-a}&plus;\frac{1}{1-a}\frac{\sigma&space;\sqrt{2T}}{\sqrt{\pi&space;}})

即可算出![](http://latex.codecogs.com/svg.latex?K_{0})的值。由![](http://latex.codecogs.com/svg.latex?K_{0})的值，通过下面的线性公式得到![](http://latex.codecogs.com/svg.latex?K)的值。

![](http://latex.codecogs.com/svg.latex?K^{'}=\gamma*K_{0})

可设定：![](http://latex.codecogs.com/svg.latex?\gamma=0.5&space;)，注意![](http://latex.codecogs.com/svg.latex?\gamma)是可调整的参数，![](http://latex.codecogs.com/svg.latex?0<\gamma<1)。

此外，在实践中为了更节省成本，考虑计算所有![](http://latex.codecogs.com/svg.latex?(T,\sigma&space;))。对的![](http://latex.codecogs.com/svg.latex?K_{0})平均值。计算公式如下：

![](http://latex.codecogs.com/svg.latex?\bar{K}_{0}=E[K_{0}(T,\sigma)])

这里的![](http://latex.codecogs.com/svg.latex?\bar{K}_{0})就是![](http://latex.codecogs.com/svg.latex?K_{0})在随机向量![](http://latex.codecogs.com/svg.latex?(T,\sigma&space;))联合分布下的期望。

基于2020年7月13日到2020年9月13日的数据，我们估计得到![](http://latex.codecogs.com/svg.latex?K_{0})的期望值![](http://latex.codecogs.com/svg.latex?\bar{K}_{0}\approx0.005)

通过![](http://latex.codecogs.com/svg.latex?\bar{K}_{0})来调整执行价格，做市商所面临的价格偏差（由于延时![](http://latex.codecogs.com/svg.latex?T)和波动率![](http://latex.codecogs.com/svg.latex?\sigma)给带来的风险），在长时间内应能得到充分完全的补偿。

#### 冲击成本

当CoFiX资产池规模足够大的时候，单笔交易或者单位时间累积交易很难达到资产的上限，但单笔的大交易量可能影响做市商的对冲成本。因此单位时间内数额较大的交易应当支付一定的价差，此价差称之为冲击成本C（给定交易量对价格的影响）。不考虑冲击成本时的K值，加上冲击成本，得到对做市商的价差补偿系数![](http://latex.codecogs.com/svg.latex?K_{m})的值：

![](http://latex.codecogs.com/svg.latex?K_{m}=K&plus;C)

由于做市商可以在全市场的所有交易所对冲交易，因此这里的冲击成本要考虑全市场的交易情况，因此在一般的交易行为下，不太会发生，只有巨额交易才需要，其规模取决于交易所的交易深度，可以用如下线性公式来估计冲击成成本：

![](http://latex.codecogs.com/svg.latex?C=\alpha&plus;\beta*VOL)

其中VOL为交易的手数，我们基于前10大数字货币交易所买卖ETH的深度行情数据估计出的结果如下：

买入时：![](http://latex.codecogs.com/svg.latex?\alpha=2.570e-05,\beta=8.542e-07)

卖出时：![](http://latex.codecogs.com/svg.latex?\alpha=-1.171e-04,\beta=8.386e-07)

交易量较小时冲击成本可以忽略不计，C的计算公式当前设定如下：

![](http://latex.codecogs.com/svg.latex?c=%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D0%20&%20VOL%3C500%20%5C%5C%5Calpha%20&plus;%5Cbeta%20*VOL%20&%20VOL%5Cgeq%20500%20%5C%5C%5Cend%7Bmatrix%7D%5Cright.)

### 七、其他问题

1. 当K值实时更新时，由于K值的计算受多种因素影响，可能导致极不合理的价格，从而给系统或者实际用户造成损失，规定停机条件如下：如果![](http://latex.codecogs.com/svg.latex?K_{0})值大于5%，停机。如果使用期望值![](http://latex.codecogs.com/svg.latex?\bar{K}_{0}) 来对做市商进行补偿，则不存在![](http://latex.codecogs.com/svg.latex?K_{0})的停机条件。

2. K值的计算过程较复杂，会消耗较多的gas，因此需要对K值缓存，在一段时间内，采用之前已经计算出来的K值，具体缓存时间作为一个参数配置，初始值需要根据经验确定。
