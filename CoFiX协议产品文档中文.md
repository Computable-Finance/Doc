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

这里的买入和卖出都是相对价格P所对应的本位资产来说的，例如价格P是以ETH为本位的， 即一单位ETH等于多少USDT、HBTC等资产，则买入和卖出指的是买入和卖出ETH（卖出时其实就是买入本位资产所对应的另一方资产）。

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

2. 使用 x 个  ETH 兑换出 y 个 HBTC

![](http://latex.codecogs.com/svg.latex?y=x*P_{2,s}^{'}*(1-\theta&space;))

3. 1 和 2步骤合并可以得出，使用 a 个 USDT 可以兑换出的 HBTC数量为

![](http://latex.codecogs.com/svg.latex?y=(a/P_{1,b}^{'})*P_{2,s}^{'}*(1-\theta&space;)^{2})

其中：

![](http://latex.codecogs.com/svg.latex?P_{1})为ETH/USDT预言机价格, ![](http://latex.codecogs.com/svg.latex?K_{1})为ETH:USDT交易池价格补偿系数;![](http://latex.codecogs.com/svg.latex?P_{2})为ETH/HBTC预言机价格 ,![](http://latex.codecogs.com/svg.latex?K_{2})为ETH:HBTC交易池价格补偿系数; ![](http://latex.codecogs.com/svg.latex?\theta)为交易手续费系数。

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

矿池B：由固定出矿和浮动出矿两部分，固定出矿部分，每个区块出矿9个，每240万个区块后衰减到上期值的80%，960万个区块后不再衰减，按照每个块出矿；浮动部分为交易出矿的10%。

矿池C：当由固定出矿和浮动出矿两部分，固定出矿部分，每个区块出矿1个，每240万个区块后衰减到上期值的80%，960万个区块后不再衰减，按照每个块出矿；浮动部分为交易出矿的10%。

### 7.2、交易者挖矿模型：

交易者每笔挖矿都会产出一定的CoFi，其中80%归该交易者，10%归节点矿池，10%归做市商矿池。每笔产出的CoFi主要取决于该笔交易支付的佣金：

1. 假设该笔交易的规模为![](http://latex.codecogs.com/svg.latex?X_{t})(ETH)，佣金为![](http://latex.codecogs.com/svg.latex?Y_{t}=X_{t}*\theta&space;)(ETH)；
2. 单位佣金（1ETH）挖出的CoFi标准量为at  , at的计算公式如下：

![](http://latex.codecogs.com/svg.latex?a_{t}=\frac{b_{t}*\varphi*2400000}{X_{t}*N_{p}*0.3})

公式注释：

(1) ![](http://latex.codecogs.com/svg.latex?b_{t})为做市商矿池在当前时刻的单位区块出矿量；

(2) ![](http://latex.codecogs.com/svg.latex?X_{t})为当前交易对池子的总份额；

(3) ![](http://latex.codecogs.com/svg.latex?N_{p})为当前交易对池子的净值；

(4) ![](http://latex.codecogs.com/svg.latex?\varphi)为对应交易对的出矿系数，也即当前交易对池子的出矿占比。

3. 考虑到连续若干笔交易规模过大，会导致出矿不受控制，因此我们设计了密度衰减指标，其核心参数如下：

(1) 单笔交易触发密度衰减的阈值为![](http://latex.codecogs.com/svg.latex?L*\theta*a_{t})，其中![](http://latex.codecogs.com/svg.latex?L)为做市商对应矿池的资产总额（ETH）的千分之一，即![](http://latex.codecogs.com/svg.latex?L=\frac{X_{t}*N_{p}}{1000})，![](http://latex.codecogs.com/svg.latex?L)的最小值为100（ETH）；

(2) 假设本次交易和上次交易之间的区块间隔为s，则密度参数:

![](http://latex.codecogs.com/svg.latex?f_%7Bt%7D=%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D%20f_%7Bt-1%7D*(300-s)/300&plus;y_%7Bt%7D*a_%7Bt%7D&s%5Cleq%20300%5C%5C%20y_%7Bt%7D*a_%7Bt%7D&s%3E%20300%5C%5C%5Cend%7Bmatrix%7D%5Cright.)

其中：![](http://latex.codecogs.com/svg.latex?f_{0}=0)，本次交易和上次交易在同一区块或本次交易是第一笔交易时![](http://latex.codecogs.com/svg.latex?s=0)；

(3) 为了确保交易池两端资产的平衡，在挖矿中加入一个平衡系数![](http://latex.codecogs.com/svg.latex?\lambda&space;)，![](http://latex.codecogs.com/svg.latex?\lambda&space;)计算如下：

假设交易者用资产![](http://latex.codecogs.com/svg.latex?V_{x})兑换资产![](http://latex.codecogs.com/svg.latex?V_{y})， 交易池内资产![](http://latex.codecogs.com/svg.latex?V_{x})总量为![](http://latex.codecogs.com/svg.latex?U_{x})，资产![](http://latex.codecogs.com/svg.latex?V_{y})总量为![](http://latex.codecogs.com/svg.latex?U_{y})（按交易时的价格换算成ETH），![](http://latex.codecogs.com/svg.latex?\lambda&space;)的值的大小取决于![](http://latex.codecogs.com/svg.latex?U_{x}/U_{y})的值的大小，具体公式如下： 

![](http://latex.codecogs.com/svg.latex?%5Clambda=%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D0.50%20&%20U_%7Bx%7D/U_%7By%7D%5Cgeq%2010%20%5C%5C0.75%20&%2010%3EU_%7Bx%7D/U_%7By%7D%5Cgeq%203%20%5C%5C1.00%20&%203%3EU_%7Bx%7D/U_%7By%7D%5Cgeq%200.33%20%5C%5C1.33%20&%200.33%3EU_%7Bx%7D/U_%7By%7D%5Cgeq%200.1%20%5C%5C2.0%20&%20U_%7Bx%7D/U_%7By%7D%3C%200.1%20%5C%5C%5Cend%7Bmatrix%7D%5Cright.)

(4) 出矿量公式：

![](http://latex.codecogs.com/svg.latex?y_{t})对应的出矿量![](http://latex.codecogs.com/svg.latex?A(y_{t}))计算如下：

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

(1) 为矿池B在![](http://latex.codecogs.com/svg.latex?h_{t})时刻的单位区块固定部分的出矿量,初始![](http://latex.codecogs.com/svg.latex?b_{0}=9)，过240万个区块衰减到上期值的80%，960万个区块后不再衰减，按照每个块出矿；

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

(1) ![](http://latex.codecogs.com/svg.latex?c_{t})为矿池C在![](http://latex.codecogs.com/svg.latex?h_{t})时刻的单位区块固定出矿部分，每个区块出矿1个，每240万个区块后衰减到上期值的80%，960万个区块后不再衰减，按照每个块出矿；

(2) ![](http://latex.codecogs.com/svg.latex?\Sigma&space;R(y_{t}))为![](http://latex.codecogs.com/svg.latex?h_{t-1})到![](http://latex.codecogs.com/svg.latex?h_{t})期间交易池交易挖矿分给节点的部分；

(3) ![](http://latex.codecogs.com/svg.latex?N_{t})为t时刻存入节点矿池的节点token总余额。

### 7.5、CoFi分红及回购模型：

参与挖矿的交易池，其所有交易佣金ETH进入系统分红池，其中ɑ比例用于分红，1-ɑ用于回购，回购机制在CoFi上线CoFiX后由DAO另行设计。

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

第一阶段，由 10个多签超级管理员地址来主导，负责对 CoFiX 早期阶段进行合约升级、参数调整。管理员地址主要由社区 KOL、早期投资人、合作项目方等多方组成。

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

