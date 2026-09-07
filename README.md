# X402Gate

**Makes the pool an x402 resource server: a swapper that presents a signed x402 payment gets a cheaper fee on that swap, and the payment settles atomically with the trade.**

A production Uniswap v4 hook. It prices every swap by overriding the pool's LP fee, so the value it captures is paid to in-range liquidity and never to the hook. No owner, no pause switch, no upgrade path.

- **Site:** https://x402-gate.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/X402GateHook.sol`](src/hooks/X402GateHook.sol)
- **Licence:** Apache-2.0

## How it works

x402 is how autonomous agents already pay for things. An agent asks for a resource, gets back HTTP 402 with a machine-readable price, signs an EIP-3009 authorization for that amount, retries with the signature in a header, and the resource server settles it. Thousands of endpoints speak it and every agent framework has a client for it.

What nothing speaks it for is liquidity, which is odd, because a fee tier is exactly the kind of thing an agent would want to buy: it is metered, it is worth different amounts to different callers, and the caller knows its own value better than the venue does. The usual way to give one class of swapper a better price is to gate on what they *are*: hold this NFT, be on this allowlist, pass this KYC check. Every one of those is a proxy for willingness to pay, maintained by hand, and wrong at the edges.

This hook gates on the thing itself. Pay the pool's posted price and the swap is cheaper. Do not, and it is not.

Nobody curates anything. Three properties fall out of doing this in the hook rather than over HTTP: The 402 challenge is on-chain. {quote} returns the same fields an x402 client reads out of a 402 response body: scheme, network, amount, asset, recipient, resource, timeout.

An agent discovers the price with one `eth_call` against the pool it was already going to trade, with no endpoint to find, no server to be up, and no TLS. Payment and delivery are atomic, which over HTTP they are not. An x402 client that pays for an API call and then receives a 500 has paid for nothing and must argue about a refund.

Here the payment settles inside `beforeSwap`, so if the swap reverts for any reason afterwards, slippage, liquidity, another hook, the payment reverts with it. The failure mode that makes x402 awkward to build on does not exist in this direction. There is no facilitator.

The x402 deployment model puts a trusted service between payer and resource server to verify and broadcast the authorization. The pool can do both itself, because it is already a contract and the payment is already an on-chain object, so the trusted third party is simply absent rather than decentralized. How the discount is applied matters.

The fee this hook returns is an *LP* fee, so the discount is paid for by liquidity providers taking less on that swap, and the payment is what compensates them. Providers are therefore selling cheap execution for a fixed fee up front instead of a variable one on the back end, which is a trade they can price: if the posted price is set below the fee revenue given up, the pool leaks, and setting it is the one judgement the pool creator has to get right. Payments are pulled with `receiveWithAuthorization` rather than `transferWithAuthorization`.

Both are EIP-3009 and x402's own reference facilitator uses the latter, but the latter can be broadcast by anyone: a bystander could submit the payer's authorization on its own, the payment would land, the nonce would burn, and the swap that the payment was for would then revert with the payer out of pocket. sender == to` removes that. Unrecognised `hookData` is ignored rather than rejected.

The payload is prefixed with {X402_PAYMENT_MAGIC}, and anything that does not start with it is treated as "no payment offered" and charged the base fee. A pool that reverted on hookData it did not understand would be untradeable through any router that puts its own data there, which is most of them.

## Prior art

Fee discounts gated on NFT or token ownership, allowlists and KYC attestations are among the most common hook patterns, and x402 itself is a widely deployed HTTP payment protocol with an EIP-3009 settlement scheme. Neither has met the other: making an AMM pool an x402 resource server, so that the 402 challenge is an `eth_call` and settlement is atomic with the swap it paid for, is the contribution here.

## Where it does not help

A payment costs a full EIP-3009 transfer, so the discount only repays its own gas above a swap size that depends on the chain and the spread between the two fee tiers. On a mainnet-priced chain that floor is high enough that this is a hook for agents moving real size, not for retail swaps. It also inherits the hookData problem: an aggregator that does not forward hookData cannot present a payment, and its users silently pay the base fee rather than getting an error telling them why.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    X402GateHook.Config({
        asset: /* address */ 0,
        payTo: /* address */ 0,
        price: /* uint256 */ 0,
        baseFee: /* uint24 */ 0,
        discountedFee: /* uint24 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```

The pool's `fee` field must be `LPFeeLibrary.DYNAMIC_FEE_FLAG`. The hook rejects a pool initialized without it.

### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `asset` | `address` |  |
| `payTo` | `address` |  |
| `price` | `uint256` |  |
| `baseFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `discountedFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `DiscountNotADiscount(uint24,uint24)` | A pool posted a discounted fee that is not below its base fee, so paying would buy nothing. |
| `FeeTooLarge(uint24)` | A fee was configured above the protocol maximum of 100%. |
| `InvalidTerms()` | Terms named the zero address as the payment asset or the payee. |
| `NotDynamicFee()` | The hook was attempted to be initialized with a non-dynamic fee. |
| `NothingCollected()` | There is nothing accrued for this pool to withdraw. |
| `PaymentBelowPrice(uint256,uint256)` | The payment offered is smaller than the pool's posted price. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `PriceRequired()` | Terms posted a price of zero, which would make the discount free and the gate pointless. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `beforeSwap`

Mask: `0x1080`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # X402Gate
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # agent-native, x402, dynamic-fee, metering
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/x402-gate
cd x402-gate
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
