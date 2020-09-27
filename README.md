# CoFiX Product Documentation

## 1. Introduction

This is the design document of an implementation of the AMM model described in the white paper "[CoFiX, A computable financial transaction model](https://cofix.io/doc/CoFiX_White_Paper.pdf)". It describes mechanisms such as price calculation, trading, market making, exception handling and front-end interaction.

## 2. System role

| Role | Definition |
| :--- | :--- |
| Trader | The party that buys/sells on CoFiX and can be either an EOA or smart contract |
| Market maker | The party that participates in CoFiX market making is the counterpart of the trader. They provide liquidity for trades and can be either an EOA or smart contract. |
| Governor | The party that participates in the governance of the CoFiX system are the CoFiX governance token holders. The CoFiX system is modified and upgraded by voting with these tokens. In the early days it was governed by the developer admin account. |

## 3. Price calculation mechanism 

### 3.1 Price source

The prices on the CoFiX exchange come from the NEST oracle. Each asset pool corresponds to a NEST oracle price pair.

Examples:

ETH-USDT asset pool, the price is from [NEST's ETH/USDT oracle](https://github.com/NEST-Protocol/NEST-oracle-V3).

### 3.2 Price compensation coefficient K

The risk of using a decentralized oracle like NEST is caused by the deviation of oracle prices and the delay between the current block and the block with the latest effective NEST price. CoFiX needs to compensate for this risk when quoting prices from NEST to ensure that the market maker is sufficiently incentivized to continue making the market. [Compensation factor](https://cofix.io/doc/Trading_Compensation_CoFiX.pdf) K is the coefficient related to the volatility rate δ and delay T. When a trader makes a transaction, he does not directly use price P but rather P' =P\*\(1+K\) \(or P' =P\*\(1-K\)\).

![](.gitbook/assets/image.png)

Similarly, when a market maker enters and exits the market, they use the price variable P' instead of P. This price is known as the transaction price.

The detailed calculation for this is shown in the [Trading Compensation of CoFiX](https://cofix.io/doc/Trading_Compensation_CoFiX.pdf).

### 3.3 Estimated price and execution price

Since a transaction in Ethereum requires a certain waiting time from initiation to execution, and the price of the NEST oracle is always in the updated state, there will be a certain difference between the price when the transaction is initiated and executed. Here we define the estimated price and execution price.

<table>
  <thead>
    <tr>
      <th style="text-align:left"></th>
      <th style="text-align:left"></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">Estimated price: <b>P<sub>es</sub></b>
      </td>
      <td style="text-align:left">
        <p>The reference price seen by the user or market maker when initiating a
          trade is taken from the latest historical price from the NEST oracle.</p>
        <p>This price is used by the front-end to display trading prices. This calculation
          factors in changes to the subscription pool share and the amount of assets
          to be redeemed.</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">Execution price: <b>P<sub>ex</sub></b>
      </td>
      <td style="text-align:left">When the transaction is executed, the latest price of the NEST oracle
        is used in the trade.</td>
    </tr>
  </tbody>
</table>

The detailed calculation method for this is described below.

**P<sub>d</sub> = \(P<sub>ex</sub> - P<sub>es</sub>\)/P<sub>es</sub>**

The transaction will revert if **P<sub>d</sub>&gt;1%**.

## 4. Market maker mechanism

Market makers are the liquidity providers of CoFiX and earn income by providing liquidity for an asset pool. CoFiX supports unilateral asset market making, and handles asset management through the **net worth** and **share** of the fund in the corresponding asset pool.

### 4.1 Market maker share

**The XToken** represents the proportion of the asset pool owned by a market maker. As market makers put assets into the pool, they receive corresponding XTokens. When a market maker exits the pool, they do so by redeeming their XToken which entitles them to their share of the pool. XToken names follow X<sub>T1</sub>, X<sub>T2</sub>, X<sub>T3</sub>...

### 4.2 Asset pool creation and initialization

Anyone can choose a price pair from the NEST oracle to create a corresponding CoFiX asset pool.

Taking ETH-USDT as an example, when the market maker pool is created:

**N<sub>p</sub>** is the net worth of each share\(XToken\), it is represented by its ETH value: initially N<sub>p</sub>=1

The quantity of ETH market-making assets =**A<sub>e</sub>**

The quantity of USDT market-making assets =**A<sub>u</sub>**

The initial amount of share\(XToken\) is S<sub>0</sub>: 

![](.gitbook/assets/screen-shot-2020-09-26-at-2.18.32-pm.png)

### 4.3 Market maker net worth of each share\(XToken\), N<sub>p</sub>

**N<sub>p</sub>** is The net worth of each share\(XToken\) in an asset pool, it is represented by its ETH value.

Whenever there is a transaction, subscription, or redemption operation; N<sub>p</sub> is updated.

Taking the ETH-USDT asset pool as an example:

ETH-USDT asset pool total issuance share is **S \(The total amount of XToken\)**

The formula for calculating the asset pool’s net worth **N<sub>p</sub>** is:

![](.gitbook/assets/image%20%288%29.png)

### 4.4 Share\(XToken\) subscription

Subscription means participating in an asset pools market making and results in obtaining XToken of that pool.

Taking the ETH-USDT asset pool as an example:

Alice subscribes amount **a** of ETH to the ETH-USDT asset pool, and receives amount **s<sub>1</sub>** of the XToken.

![](.gitbook/assets/image%20%2810%29.png)

Similarly, if Alice subscribes amount **b** of USDT to the ETH-USDT asset pool, she receives amount **s<sub>2</sub>** of the XToken.

![](.gitbook/assets/image%20%2812%29.png)

### 4.5 Share redemption

Redemption means exiting a certain asset pool by redeeming the corresponding XTokens of that pool. Redemption entitles the market maker to receive either asset of a pair from the asset pool based on their share of the pool.

Taking the ETH-USDT asset pool as an example:

θ represents an extra transaction fee

c Alice redeems amount **c** of the XToken

Then the amount **e** of ETH that can be redeemed is:

![](.gitbook/assets/image%20%2813%29.png)

The amount **u** of USDT that can be redeemed is:

![](.gitbook/assets/image%20%282%29.png)

## 5. Trading mechanism

### 5.1 ETH to ERC-20 transaction

Using ETH to exchange USDT and USDT to exchange ETH as an example:

Alice uses amount **a** of USDT to exchange amount **x** of ETH, the calculation is:

![](.gitbook/assets/image%20%287%29.png)

Alice uses amount **b** of ETH to exchange amount **y** of USDT, the calculation is:

![](.gitbook/assets/image%20%285%29.png)

### 5.2 ERC-20 to ERC-20 transactions

When there are multiple asset pools, a trader can complete the ERC-20 to ERC-20 exchange through a single transaction by calling 2 ETH-ERC-20 asset pools.

The calculation takes the process of Alice using USDT to exchange DAI as an example:

Suppose at this time

ETH/USDT oracle price = **P<sub>1</sub>**

ETH/USDT asset pool compensation factor =**K<sub>1</sub>**

ETH/DAI oracle price = **P<sub>2</sub>**

ETH/DAI asset pool compensation factor =**K<sub>2</sub>**

Then, when Alice uses amount **a** of USDT to exchange amount **y** of DAI, the calculation is:

Using amount **a** of USDT to exchange amount **x** of ETH

![](.gitbook/assets/image%20%289%29.png)

Using amount **x** of ETH to exchange amount **y** of DAI

![](.gitbook/assets/image%20%286%29.png)

Combine Steps 1 and 2

![](.gitbook/assets/image%20%284%29.png)

## 6. Additional risk control

### 6.1 Transaction delay

Considering that Ethereum has transaction congestion, the transaction may not be confirmed for a long time.

Therefore, after a transaction is initiated, there may be a large deviation between the settlement price and the estimated price when the transaction is initiated. Therefore, a variable is introduced, the transaction effective time **t**

1. When the transaction is initiated, the time is **t<sub>0</sub>**
2. When the transaction is successfully confirmed, the time is **t<sub>1</sub>**
3. Then, if **t<sub>1</sub>-t<sub>0</sub>  &gt; t** , the transaction will revert.
4. In the current setting, **t=600s**

### 6.2 Circuit breakers

According to the [Trading Compensation of CoFiX](https://cofix.io/doc/Trading_Compensation_CoFiX.pdf), if the volatility rate rises to an extreme level or the NEST system is attacked, CoFiX needs to activate an emergency procedure to protect both the trader and market maker.

The system should be able to trigger Circuit breakers when the following conditions are met:

1. The K<sub>0</sub> value exceeds a range, K<sub>0</sub>&gt;5%
2. The volatility rate σ rises to a limit, σ&gt;0.1% per second
3. Delay T exceeds a range, T&gt;900s 

## 7. Token mining incentive system

Dividend and Governance,  produced through liquidity mining.

### **7.1 G**eneral parameters**:**

#### **G**eneral Token Generation: 

#### 0% pre-mined, 100% generated through mining

**CoFi Tokens are generated through the 3 mining pools:**

* **Mining pool A, the trading mining pool:** The token amount generated per TX is based on the extra fees collected from that TX, the token generation speed \(Token per block\) of the mining pool B, the total amount of the corresponding asset pool XToken participate in mining, and the N<sub>p</sub> \(net worth per share of the corresponding asset pool\) 
* **Mining pool B, the liquidity mining pool: b<sub>t</sub>** is the ****amount of CoFi tokens generated per block, b<sub>t</sub> ≥1, start with b<sub>0</sub>=4, the amount will be reduced by 20% for every 2,400,000 blocks and kept the integer only. For example b<sub>1</sub>=3, b<sub>2</sub>=2, b<sub>3</sub>=1, b<sub>4</sub>=1, b<sub>5</sub>=1, b<sub>6</sub>=1 ...
* **Mining pool C, the node mining pool: c<sub>t</sub>** is the amount of CoFi tokens generated per block, c<sub>t</sub> = b<sub>t</sub>/9

**General token distribution:**

* 80% of the tokens generated from **Mining pool A** goes to traders, 10% goes to the liquidity providers

   of the corresponding asset pool, 10% goes to the nodes.

* 100% of the tokens generated from **Mining pool B** goes to liquidity providers, distributed evenly per asset pool, liquidity providers can claim the tokens from the liquidity mining rewards pool based on the XToken amount they put in for liquidity mining.
* 100% of the tokens generated from **Mining pool C** goes to the nodes. Nodes can claim the tokens from the node rewards pool based on the node token amount they are holding. So the nodes always receive 10% of the total token generated through the 3 mining pools.

### **7.2 Details about the trading mining:**

#### **Token generation** mechanism**:** 

In the TX of every trade,  A(y<sub>t</sub>) amount of token is generated.

For trading TX t:

* The trading volume is X<sub>t</sub> ETH, the extra fees collected from that TX t is Y<sub>t</sub>=X<sub>t</sub>*θ ETH
* The base amount of CoFi tokens generated for TX t is a<sub>t</sub>

![](.gitbook/assets/screen-shot-2020-09-26-at-3.22.50-pm.png)

* **b<sub>t</sub>** is the token generation speed \(Token per block\) of the mining pool B
* **X<sub>t</sub>** is the total amount of the corresponding asset pool XToken participate in mining
* **N<sub>p</sub>** is The net worth of each share\(XToken\) in an asset pool, it is represented by its ETH value.
* The actual amount of CoFo token generated for TX t is A(y<sub>t</sub>) 

![](.gitbook/assets/screen-shot-2020-09-26-at-3.50.43-pm.png)

* The block interval between the trading TX t and the previous trading TX is s, the token density parameter is f<sub>t</sub>

![](.gitbook/assets/screen-shot-2020-09-26-at-3.56.05-pm.png)

* L is a threshold parameter, L = 100
* λ is a balance parameter to further help with the asset pool balance:

![](.gitbook/assets/screen-shot-2020-09-26-at-4.02.59-pm.png)

* U<sub>x</sub> is the total asset value \(in ETH\) for asset X in trading pair X-Y, U<sub>y</sub> is the total asset value \(in ETH\) for asset Y in trading pair X-Y

#### **Token distribution** mechanism**:** 

For trading TX t:

* 80% A(y<sub>t</sub>) is distributed to the trader in the same trading TX
* 10% A(y<sub>t</sub>) = I<sub>j</sub>\(y<sub>t</sub>) is distributed to the corresponding liquidity mining rewards pool for liquidity providers to claim
* 10% A(y<sub>t</sub>) = R(y<sub>t</sub>) is distributed to the node rewards pool for nodes to claim

### **7.3** Details about the liquidity mining:

#### Token generation mechanism:

* b<sub>t</sub> is the amount of CoFi tokens generated per block, b<sub>t</sub> ≥1, start with b<sub>0</sub>=4, the amount will be reduced by 20% for every 2,400,000 blocks and kept the integer only. For example b<sub>1</sub>=3, b<sub>2</sub>=2, b<sub>3</sub>=1, b<sub>4</sub>=1, b<sub>5</sub>=1, b<sub>6</sub>=1 ...

#### **Token distribution** mechanism**:** 

The market maker **m** claims CoFi token from the liquidity mining reward pool **j** through TX **t**:

* The CoFi token claim can be trigger by either claim TX, XToken deposit, or XToken withdraw TX. 
* The amount of CoFi token can be claimed through TX t is B<sub>mt</sub>(x<sub>mt-1</sub>)

![](.gitbook/assets/screen-shot-2020-09-26-at-4.55.44-pm.png)

* **x<sub>mt-1</sub>** is the liquidity provider's XToken balance in liquidity mining reward pool **j** before TX t  ****
* **G<sub>mt</sub>** is the mining factor for TX t, it represents how many CoFi the liquidity provider m can claim per XToken. **G<sub>mt</sub>t** is read from the system variable **G<sub>t</sub>**
* **G<sub>mt-1</sub>** is the mining factor for previous TX done by the liquidity provider m

![](.gitbook/assets/screen-shot-2020-09-26-at-6.08.12-pm.png)

* There are amount **n** of asset pools participate in liquidity mining, the corresponding Xtoken are X<sub>T1</sub>, X<sub>T2</sub>... X<sub>Tn</sub>. 
* G<sub>0</sub> = 0 
* h<sub>t</sub> is the time of the TX t, h<sub>t-1</sub> is the time of the previous claim, deposit, or withdraw TX
* The summation interval of ∑I<sub>j</sub>(Y<sub>i</sub>) is h<sub>t</sub> to h<sub>t-1</sub>
* X<sub>t</sub> is the total amount of XToken participate in the liquidity mining reward pool j when TX t is submitted 
* I<sub>j</sub>\(y<sub>i</sub>) is the amount of CoFi token received from the trading mining pool during between h<sub>t-1</sub> to h<sub>t</sub>

### **7.4** Details about the node mining:

#### Token generation mechanism:

* c<sub>t</sub> is the amount of CoFi tokens generated per block, c<sub>t</sub> = b<sub>t</sub>/9

#### Token **distribution** mechanism:

* There are in total of 100 nodes\(100 CoFiX node tokens\)
* Nodes deposit CoFiX node tokens in node mining to claim CoFi tokens
* The CoFi token claim can be trigger by either a claim TX, a CoFiX node tokens deposit or XToken withdraw TX.

The node **m** claims CoFi token from the node reward pool through TX **t**:

* The amount of CoFi token can be claimed through TX t is c<sub>mt</sub>(n<sub>mt-1</sub>)

![](.gitbook/assets/screen-shot-2020-09-26-at-7.26.28-pm.png)

* h<sub>t</sub> is the time of the TX t, h\_t-1 is the time of the previous claim, deposit, or withdraw TX
* n<sub>mt-1</sub> is m's balance of CoFiX node tokens in the node mining pool before TX t
* **D<sub>mt</sub>** is the mining factor for TX t, it represents how many CoFi token the node m can claim per node token. D<sub>mt</sub> is read from the system variable D<sub>t</sub>
* **D<sub>mt-1</sub>** is the mining factor for previous TX done by the node m

![](.gitbook/assets/screen-shot-2020-09-26-at-6.59.08-pm.png)

* The summation interval of ∑R(y<sub>t</sub>) is h<sub>t</sub> to h<sub>t-1</sub>
* R(y<sub>t</sub>) is the amount of CoFi token received from the trading mining pool during between h<sub>t-1</sub> to h<sub>t</sub>
* N<sub>t</sub> is the total amount of XToken participate in the node mining when TX t is submitted

### 7.5 CoFi token dividend and repurchase model：

All the extra fees \(in ETH\) collected from trading go in a saving pool,  the percentage a of the total amount will be used for dividends, the percentage 1-a of the total amount will be used for repurchase. 

#### The repurchase mechanism\(TBC by DAO\)

#### The dividend mechanism:

* CoFi token holders deposit CoFi  tokens in the dividend pool to claim CoFi tokens
* The ETH dividend claim can be trigger by either a claim TX, a CoFi token deposit TX, or a CoFi token withdraw TX.

CoFi token holder m makes a dividend claiming TX t:

* The total amount of ETH can be claimed in TX t is E<sub>mt</sub>(n<sub>mt-1</sub>)

![](.gitbook/assets/screen-shot-2020-09-26-at-7.11.16-pm.png)

* n<sub>mt-1</sub> is m's balance of CoFi tokens in the dividend pool before TX t
* **F<sub>mt</sub>** is the dividend factor for TX t, it represents how many ETH the CoFi token holder m can claim per CoFi token. F<sub>mt</sub> is read from the system variable F<sub>t</sub>
* **F<sub>mt-1</sub>** is the dividend factor for previous TX done by the CoFi token holder m

![](.gitbook/assets/screen-shot-2020-09-26-at-7.18.03-pm.png)

* F<sub>0</sub> = 0
* ∑y is the total amount of extra fees collected to the saving pool between h<sub>t</sub> to h<sub>t-1</sub>
* N<sub>t</sub> is the total amount of CoFi token participate in the dividend pool when TX t is submitted

## 8. CoFiX DAO \(TBC\)



