# Pancakeswap Vertex


### Example Queries 
Each of the queries enables a set of views for the entity, with a larger DApp specific view to represent the state of the ecosystem. These views are constructed through a generalized set of query views. 

- Query for Ecosystem

```graphql
pancake {
volume
transactions
...
}

```

- Top 100 Tokens 

```graphql
tokens(descending: true, orderBy: "volume", first: 100) {
...
}
```

- Top 100 Pairs with BNB as Base
```graphql
pairs(where: {name: "baseAsset", exp: "=", value: "BNB"}, descending: true, orderBy: "volume", first: 100) {
...
}

```


## Queries 

Each entity can be queried through the use of individual ID queries, and from a generalized query. 


```graphql
tokens(where: {name: "name", exp: "=", value: "BNB"}, descending: boolean, orderBy: string, first: int, last: int,  limit: int) {
 [Tokens]
}

token(id string) {
Token
}
```

## Entities 


```graphql
type PancakeFactory @entity {
  # factory address
  id: ID!

  # pair info
  pairCount: Int!

  # total volume
  totalVolumeUSD: BigDecimal!
  totalVolumeBNB: BigDecimal!

  # untracked values - less confident USD scores
  untrackedVolumeUSD: BigDecimal!

  # total liquidity
  totalLiquidityUSD: BigDecimal!
  totalLiquidityBNB: BigDecimal!

  # transactions
  txCount: BigInt!
}

type Token @entity {
  # token address
  id: ID!

  # mirrored from the smart contract
  symbol: String!
  name: String!
  decimals: BigInt!

  # used for other stats like marketcap
  totalSupply: BigInt!

  # token specific volume
  tradeVolume: BigDecimal!
  tradeVolumeUSD: BigDecimal!
  untrackedVolumeUSD: BigDecimal!

  # transactions across all pairs
  txCount: BigInt!

  # liquidity across all pairs
  totalLiquidity: BigDecimal!

  # derived prices
  derivedBNB: BigDecimal

  # derived fields
  tokenDayData: [TokenDayData!]! @derivedFrom(field: "token")
  pairDayDataBase: [PairDayData!]! @derivedFrom(field: "token0")
  pairDayDataQuote: [PairDayData!]! @derivedFrom(field: "token1")
  pairBase: [Pair!]! @derivedFrom(field: "token0")
  pairQuote: [Pair!]! @derivedFrom(field: "token1")
}

type Pair @entity {
  # pair address
  id: ID!

  # mirrored from the smart contract
  token0: Token!
  token1: Token!
  reserve0: BigDecimal!
  reserve1: BigDecimal!
  totalSupply: BigDecimal!

  # derived liquidity
  reserveBNB: BigDecimal!
  reserveUSD: BigDecimal!
  trackedReserveBNB: BigDecimal! # used for separating per pair reserves and global
  # Price in terms of the asset pair
  token0Price: BigDecimal!
  token1Price: BigDecimal!

  # lifetime volume stats
  volumeToken0: BigDecimal!
  volumeToken1: BigDecimal!
  volumeUSD: BigDecimal!
  untrackedVolumeUSD: BigDecimal!
  txCount: BigInt!

  # creation stats
  createdAtTimestamp: BigInt!
  createdAtBlockNumber: BigInt!

  # derived fields
  pairHourData: [PairHourData!]! @derivedFrom(field: "pair")
  mints: [Mint!]! @derivedFrom(field: "pair")
  burns: [Burn!]! @derivedFrom(field: "pair")
  swaps: [Swap!]! @derivedFrom(field: "pair")
}

type Transaction @entity {
  id: ID! # txn hash
  blockNumber: BigInt!
  timestamp: BigInt!
  # This is not the reverse of Mint.transaction; it is only used to
  # track incomplete mints (similar for burns and swaps)
  mints: [Mint]!
  burns: [Burn]!
  swaps: [Swap]!
}

type Mint @entity {
  # transaction hash + "-" + index in mints Transaction array
  id: ID!
  transaction: Transaction!
  timestamp: BigInt! # need this to pull recent txns for specific token or pair
  pair: Pair!

  # populated from the primary Transfer event
  to: Bytes!
  liquidity: BigDecimal!

  # populated from the Mint event
  sender: Bytes
  amount0: BigDecimal
  amount1: BigDecimal
  logIndex: BigInt
  # derived amount based on available prices of tokens
  amountUSD: BigDecimal

  # optional fee fields, if a Transfer event is fired in _mintFee
  feeTo: Bytes
  feeLiquidity: BigDecimal
}

type Burn @entity {
  # transaction hash + "-" + index in mints Transaction array
  id: ID!
  transaction: Transaction!
  timestamp: BigInt! # need this to pull recent txns for specific token or pair
  pair: Pair!

  # populated from the primary Transfer event
  liquidity: BigDecimal!

  # populated from the Burn event
  sender: Bytes
  amount0: BigDecimal
  amount1: BigDecimal
  to: Bytes
  logIndex: BigInt
  # derived amount based on available prices of tokens
  amountUSD: BigDecimal

  # mark uncomplete in BNB case
  needsComplete: Boolean!

  # optional fee fields, if a Transfer event is fired in _mintFee
  feeTo: Bytes
  feeLiquidity: BigDecimal
}

type Swap @entity {
  # transaction hash + "-" + index in swaps Transaction array
  id: ID!
  transaction: Transaction!
  timestamp: BigInt! # need this to pull recent txns for specific token or pair
  pair: Pair!

  # populated from the Swap event
  sender: Bytes!
  from: Bytes! # the EOA that initiated the txn
  amount0In: BigDecimal!
  amount1In: BigDecimal!
  amount0Out: BigDecimal!
  amount1Out: BigDecimal!
  to: Bytes!
  logIndex: BigInt

  # derived info
  amountUSD: BigDecimal!
}

# stores for USD calculations
type Bundle @entity {
  id: ID!
  bnbPrice: BigDecimal! # price of BNB usd
}

# Data accumulated and condensed into day stats for all of Pancake
type PancakeDayData @entity {
  id: ID! # timestamp rounded to current day by dividing by 86400
  date: Int!

  dailyVolumeBNB: BigDecimal!
  dailyVolumeUSD: BigDecimal!
  dailyVolumeUntracked: BigDecimal!

  totalVolumeBNB: BigDecimal!
  totalLiquidityBNB: BigDecimal!
  totalVolumeUSD: BigDecimal! # Accumulate at each trade, not just calculated off whatever totalVolume is. making it more accurate as it is a live conversion
  totalLiquidityUSD: BigDecimal!

  txCount: BigInt!
}

type PairHourData @entity {
  id: ID!
  hourStartUnix: Int! # unix timestamp for start of hour
  pair: Pair!

  # reserves
  reserve0: BigDecimal!
  reserve1: BigDecimal!

  # total supply for LP historical returns
  totalSupply: BigDecimal!

  # derived liquidity
  reserveUSD: BigDecimal!

  # volume stats
  hourlyVolumeToken0: BigDecimal!
  hourlyVolumeToken1: BigDecimal!
  hourlyVolumeUSD: BigDecimal!
  hourlyTxns: BigInt!
}

# Data accumulated and condensed into day stats for each exchange
type PairDayData @entity {
  id: ID!
  date: Int!
  pairAddress: Bytes!
  token0: Token!
  token1: Token!

  # reserves
  reserve0: BigDecimal!
  reserve1: BigDecimal!

  # total supply for LP historical returns
  totalSupply: BigDecimal!

  # derived liquidity
  reserveUSD: BigDecimal!

  # volume stats
  dailyVolumeToken0: BigDecimal!
  dailyVolumeToken1: BigDecimal!
  dailyVolumeUSD: BigDecimal!
  dailyTxns: BigInt!
}

type TokenDayData @entity {
  id: ID!
  date: Int!
  token: Token!

  # volume stats
  dailyVolumeToken: BigDecimal!
  dailyVolumeBNB: BigDecimal!
  dailyVolumeUSD: BigDecimal!
  dailyTxns: BigInt!

  # liquidity stats
  totalLiquidityToken: BigDecimal!
  totalLiquidityBNB: BigDecimal!
  totalLiquidityUSD: BigDecimal!

  # price stats
  priceUSD: BigDecimal!
}


type Pair @entity {
    id: ID! # address
    token0: Bytes!
    token1: Bytes!
}

type Candle @entity {
    id: ID! # time + period + srcToken + dstToken
    time: Int!
    period: Int!
    lastBlock: Int!
    token0: Bytes!
    token1: Bytes!

    token0TotalAmount: BigInt!
    token1TotalAmount: BigInt!
    open: BigDecimal!
    close: BigDecimal!
    low: BigDecimal!
    high: BigDecimal!
}

type Block @entity {
    id: ID!
    parentHash: String!
    unclesHash: String!
    author: String!
    stateRoot: String!
    transactionsRoot: String!
    receiptsRoot: String!
    number: BigInt!
    gasUsed: BigInt!
    gasLimit: BigInt!
    timestamp: BigInt!
    difficulty: BigInt!
    totalDifficulty: BigInt!
    size: BigInt
}
```
