# Salt and token ordering

## What the salt is

Clanker v4 deploys tokens with CREATE2. The CREATE2 salt is
`keccak256(abi.encode(tokenAdmin, salt))`, so for a fixed admin the `salt` field is the one
free input that determines the token's contract address. It has no other effect.

Every Clanker token has a salt. clanker.world never shows the field: its backend grinds salts
until the address ends in `…4b07` (the brand suffix, see `findVanityAddressV4` in the SDK).
That grind fixes only the last two bytes; it does not influence which side of the pool the
token lands on.

## Why the address matters at all

Uniswap V3 and V4 have no convention for which token is token0. The rule is purely numeric:
`currency0 < currency1` by address. Whichever of the two contract addresses is the smaller
integer is token0.

Consequences of the token being token1 (i.e. address above the wrapped native):

- The pool tick is mirrored: price of the token rises as the tick falls.
- Clanker's locker mirrors its position ticks automatically
  (`tickLower = -tickUpper`, `tickUpper = -tickLower`), and `PoolRaise` does the same for the
  floor and token bands. The economics are identical in both orderings.
- Some charting tools draw the pool "upside down" unless they normalise; clanker.world does.
- In code it decides which of `getLiquidityForAmount0` / `getLiquidityForAmount1` applies.

Nothing about fees, depth, the push, or the reward slots depends on it.

## Odds per chain

Wrapped-native addresses from the Clanker SDK's `WETH_ADDRESSES` table. "Native is token0"
is the probability that a random token address sorts above the wrapped native — just the
leading bytes as a fraction of the address space.

| Chain | Wrapped native | Address | Native is token0 | Clanker token is token0 |
|---|---|---|---|---|
| Robinhood | WETH (non-standard) | `0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73` | ~95% | ~5% |
| Abstract | WETH | `0x3439153EB7AF838Ad19d56E1571FBD09333C2809` | ~80% | ~20% |
| Monad mainnet | WMON | `0x3bd359c1119da7da1d913d1c4d2b7c461115433a` | ~77% | ~23% |
| Base | WETH (OP predeploy) | `0x4200000000000000000000000000000000000006` | ~74% | ~26% |
| Unichain | WETH (OP predeploy) | `0x4200000000000000000000000000000000000006` | ~74% | ~26% |
| Arbitrum | WETH | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` | ~49% | ~51% |
| BNB Chain | WBNB | `0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c` | ~27% | ~73% |
| Ethereum | WETH | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` | ~25% | ~75% |

The folk belief that "ETH is usually token1" is an Ethereum-mainnet habit. On every OP-stack
chain and on Robinhood the reverse holds. Base, Base Sepolia and Unichain share the same
predeploy address, so anything tuned for Base carries over to Unichain unchanged.

Native ETH (`address(0)`) is always token0 when a V4 pool uses it, but Clanker pools pair with
the wrapped token, never native.

## Grinding for ordering

Cost of grinding a salt until the token sorts below the wrapped native (token = token0):

| Chain | Expected tries |
|---|---|
| Ethereum | ~1.3 |
| BNB Chain | ~1.4 |
| Arbitrum | ~2 |
| Base / Unichain | ~4 |
| Monad | ~4.3 |
| Abstract | ~5 |
| Robinhood | ~20 |

All trivial off-chain (`predictTokenAddressV4` in the SDK, or the CREATE2 formula directly).
On Robinhood, though, a token0 Clanker token would be the oddity: ~95% of tokens on that chain
are token1 against WETH, and every chart and tool there is used to that orientation.

## PoolRaise policy

- Random salt by default. The field is not shown on the creator form.
- No grinding for ordering. Both orderings are handled by the contract and covered by tests
  (`test_fullLifecycle_tokenIsCurrency0` / `_tokenIsCurrency1`).
- `TOKEN_SALT` remains available on the script path for anyone who wants a vanity or
  predicted address.
- If Clanker ever allowlists the factory as a trusted deployer and wants the `…4b07` suffix,
  the UI can call `findVanityAddressV4` before `createRaise`; that is cosmetic and changes
  nothing in the contract.

Sources: [Uniswap V4 PoolKey](https://docs.uniswap.org/contracts/v4/reference/core/types/PoolKey),
[Uniswap V3 pool immutables](https://docs.uniswap.org/contracts/v3/reference/core/interfaces/pool/IUniswapV3PoolImmutables),
[clanker-sdk `src/constants.ts`](https://github.com/clanker-devco/clanker-sdk/blob/main/src/constants.ts).
