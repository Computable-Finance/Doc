
# Market-making is not for everyone

Market Making is an advanced financial activity that carries the risk of losing money when compared to just holding your assets. If you do not understand the risks or know how to hedge against asset proportion changes, we strongly recommend you learn these before adding your hard-earned crypto assets to the liquidity pools. **Otherwise, do not add liquidity.**

*Disclaimer: This content is for informational purposes only, you should not construe any such information or other material as legal, tax, investment, financial, or other advice.*

## Risks

CoFiX provides market making with no permanent or impermanent loss, but Market Makers (or “MMs” for short) can still lose money. Asset proportion changes are the main risk. These changes happen because of trader and market maker activity during market changes.

CoFiX does not control this proportion. As one cannot predict what traders and MMs will do, we only build assumptions to react on. For example, if the ETH market is on a downturn, everyone might try to cash into USDT. MMs have to hedge against this risk themselves if they wish to lock in profits.

If they do not, it’s very likely that MMs will lose money, and there will be little reason to invest in CoFiX.

## How can a Market Maker hedge?

It can be done in multiple ways. MMs can hedge by trading part of their assets in other markets, mirroring proportion changes in the liquidity pool in relation to their share of the pool. Like this, a MM can eliminate negative fluctuations and ensure that their asset position is always in a state of growth, regardless of the market price.

As a Market Maker, you have to monitor the asset pool. When you add liquidity to the CoFiX pool, you receive XTokens, which represents your percentage of the liquidity pool in ETH Net Worth.

The price of XT can change over time, which affects your liquidity returns. This can happen during (1) pool balance changes, (2) pool proportion changes or (3) price-ratio changes. Both Asset Proportion and Price Ratio needs to change for loss to occur.

*Visit this [Repository](https://github.com/Computable-Finance/CoFiX-hedger/blob/master/README.md) for a basic tool for market makers to hedge.*

## A Hedging Example

Alice decides to participate in CoFiX market making. She has 200 ETH and 2000 USDT, of which she places half (100 ETH and 1000 USDT) in CoFiX to make the market, and the other half is placed off-market for hedging. For simplicity, let’s say Alice has 100% of the pool, which means she’s alone in making the market.

Mark is a CoFiX trader. He comes in and uses his USDT to buy 1 ETH. At the time, the price of ETH was at 400 USDT. Taking into account the price spread and fees, Mark pays 410 USDT to buy 1 ETH.

With this, Alice’s CoFiX pool now has 99 ETH and 1410 USDT. Because Alice owns 100% of the pool, her total assets are now 199 ETH and 2410 USDT.

Because loss only happens when both asset proportion and price ratio changes, there is still a risk of price fluctuations. Alice begins to hedge off-market so that she can lock in a profit. Using a trading bot, she uses 400 USDT to buy 1 ETH, making her total assets as 200 ETH and 2010 USDT. She nets a profit of 10 USDT from her transactions.

As CoFiX transactions continue to happen, Alice will need to perform these hedging operations continuously if she wishes to see her assets in a growth state and maintain profitability. In a real scenario, Alice would also need to consider price availability, transaction costs, her share of the pool, along several other factors.

We cover hedging strategies in detail, and more examples, here (TO BE UPDATED).

## Additional Information

**1.  What is the Pool Balance?**

It’s the overall amount of ETH, USDT or HBTC available in the pool. When we refer to asset proportion changes, we generally refer to this concept. As this balance changes, so can your total XT value.

**2.  What is Pool Proportion?**

It’s your share of the pool in relation to other liquidity providers, in ETH or ETH value equivalent for other ERC20s. As your share percentage rises and drops, it can affect your total XT value.

**3.  What is Price-Ratio?**

It’s the change in prices in one currency vs the other currency in your added pair. Large directional changes will affect your total XT value. CoFiX operates with ETH-USDT and ETH-HBTC pairs.
