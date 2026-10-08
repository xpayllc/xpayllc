# X Pay developer starting point

[Website](https://x-pay.llc/) · [API catalog](https://www.api-xpay.com/) · [Profile](README.md)

## Choose the right interface

| Purpose | Public interface |
| --- | --- |
| Learn about X Pay | https://x-pay.llc/ |
| Discover paid resources | https://www.api-xpay.com/openapi.json |
| Inspect facilitator capabilities | https://facilitator-xpay.llc/supported |

The API service sells resources. The facilitator handles payment verification and settlement. A seller receiving a payment and a facilitator submitting settlement are separate roles, even when one operator provides both.

## Inspect before integrating

These read-only requests do not authorize a payment:

```sh
curl --fail --show-error --silent https://facilitator-xpay.llc/supported
curl --fail --show-error --silent https://www.api-xpay.com/openapi.json
```

The capability response checked on 9 October 2026 advertised x402 v2, the exact scheme, Base mainnet (`eip155:8453`) and USDC with 6 decimals. The advertised token contract was `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`, using `transferWithAuthorization`. Always read the current response; this document is not a live availability guarantee.

## Payment flow

1. Select a resource and inspect its method, parameters and price.
2. Request the resource and inspect the returned HTTP 402 payment requirements where payment is required.
3. Confirm the network, asset, recipient, amount and authorization expiry before signing with a compatible x402 client.
4. The service and facilitator process verification and settlement according to their integration. Only a successful confirmed settlement is payment evidence.
5. Keep the settlement transaction hash and resource response for reconciliation.

Use the current service documentation and a compatible x402 client for request formats. This guide does not provide an executable paid transaction or claim that the SDK repository has a published release. Never put private keys, API secrets or reusable signed authorizations in GitHub issues.

## Troubleshooting and support

- Unsupported network or token: compare the requested payment with the live capability response.
- Unexpected amount: check token decimals and raw units; 1 USDC is 1,000,000 base units.
- Request failure after signing: inspect settlement status before retrying; do not blindly repeat payment.
- Different dashboard totals: compare the same network, period, addresses and role. Seller receipts are not automatically facilitator settlement totals.

For documentation corrections, [open an issue](https://github.com/xpayllc/xpayllc/issues/new) with the public URL, expected behavior and redacted response. Contact X Pay through the [official website](https://x-pay.llc/) for service support.
