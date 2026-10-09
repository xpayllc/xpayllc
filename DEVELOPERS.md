# X Pay merchant integration guide

[Website](https://x-pay.llc/) · [Hosted guide](https://facilitator-xpay.llc/docs) · [Paid API catalog](https://www.api-xpay.com/openapi.json)

## Choose the right service

The paid API catalog sells resources. The facilitator verifies signed payment authorizations and submits settlements for merchants. These are separate roles. A seller receipt is not automatically evidence of which facilitator processed it.

## Merchant access

Contact **chris@x-pay.llc** for merchant onboarding and a merchant API key. Agree on supported networks, payload compatibility, fees and rate limits during onboarding. No self-service key issuance or published SDK release is promised by this guide.

Keep the key in a server-side secret manager or environment variable named `XPAY_MERCHANT_API_KEY`. Never put it in browser JavaScript, a public repository, screenshots or support messages. A merchant key authenticates the merchant; it is not a wallet private key and does not replace the payer's signed authorization.

Use either header:

```text
X-API-Key: YOUR_MERCHANT_API_KEY
Authorization: Bearer YOUR_MERCHANT_API_KEY
```

Choose one authentication header. Contact merchant support to revoke or replace an exposed key.

## Networks and endpoints

All paths use `https://facilitator-xpay.llc`.

| Method | Base | BNB Smart Chain | Access |
| --- | --- | --- | --- |
| GET | `/health` | `/bsc/health` | Public |
| GET | `/supported` | `/bsc/supported` | Public |
| GET | `/facilitator/status` | `/bsc/facilitator/status` | Public |
| POST | `/verify` | `/bsc/verify` | Merchant key |
| POST | `/settle` | `/bsc/settle` | Merchant key |

Base mainnet is `eip155:8453`: Circle USDC, 6 decimals, EIP-3009 authorization. BNB mainnet is `eip155:56`: Binance-Peg USDC, 18 decimals. The public API advertises Permit2; the separately observed contract below uses ERC-20 approval and `settleExact`. These routes must not be treated as interchangeable. Never reuse Base authorization fields or decimal conversion for BNB.

```sh
curl --fail --silent --show-error https://facilitator-xpay.llc/supported
curl --fail --silent --show-error https://facilitator-xpay.llc/bsc/supported
curl --fail --silent --show-error https://facilitator-xpay.llc/facilitator/status
curl --fail --silent --show-error https://facilitator-xpay.llc/bsc/facilitator/status
```

Health only proves reachability. Inspect current capabilities and settlement status before integration. Native network gas is required for settlement. Read token and Permit2 details from current capabilities/status and confirm them during onboarding.

## Request contract

Both payment endpoints accept a JSON object with `paymentPayload` and `paymentRequirements`. Preserve the merchant's original requirements and the compatible client's signed payload; do not invent a signature or change signed terms.

Conceptual structure (placeholders, not an executable authorization):

```json
{
  "paymentPayload": "REPLACE WITH THE CLIENT'S SIGNED PAYMENT OBJECT",
  "paymentRequirements": "REPLACE WITH THE MERCHANT'S ORIGINAL REQUIREMENTS OBJECT"
}
```

Each placeholder above must be an object, not a string, in the real request. Requirements identify the exact scheme, network, asset, payTo, amount in integer atomic units and timeout. Standard v2 payloads include `x402Version: 2`, accepted terms and the signed payload.

For Base EIP-3009, the inspected implementation uses `payload.signature` and `payload.authorization` with `from`, `to`, `value`, `validAfter`, `validBefore` and a bytes32 `nonce`. Values/timestamps are integer strings. Accepted terms, authorization and original requirements must agree. Confirm the deployed schema during onboarding. For BNB, use a compatible exact/Permit2 client and obtain the supported payload schema; do not adapt the Base example by changing only its network.

## Verify from your server

Save the actual request object securely as `payment-request.json`. This example verifies only; it does not submit settlement:

```sh
curl --silent --show-error \
  -X POST https://facilitator-xpay.llc/verify \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${XPAY_MERCHANT_API_KEY}" \
  --data-binary @payment-request.json
```

For BNB use `/bsc/verify`. Do not log the expanded key or signed payload. Check the response body: HTTP 200 can contain `isValid: false`. A valid verification is not a settlement receipt.

## Settle and deliver

Only after validating the payer's consent and a successful verification, submit the same request object to the matching `/settle` endpoint. This operation can move funds. Require `success: true`, confirm its network and transaction receipt, and match the recipient, asset and amount before delivering the resource.

The inspected Base implementation supports an `Idempotency-Key` header and can return HTTP 202 with `success: false` and `SETTLEMENT_PENDING`. Pending is not paid. Persist the payment state and transaction hash; reconcile before retrying. Do not assume Base idempotency semantics also apply to BNB without confirming them.

Bind every payment to your merchant account, resource/order, expected recipient, asset, amount and network. Deduplicate successful receipts. Never accept customer-supplied requirements as your source of truth.

## Errors and reconciliation

- **401:** missing or rejected merchant credentials; check onboarding and server configuration.
- **400 / invalid payload:** check the current schema, network, exact amount, expiry and authorization.
- **200 with `isValid: false`:** verification rejected; inspect the reason, do not deliver.
- **202 / pending, timeout or connection loss:** settlement may still be in progress; reconcile before retrying or asking the payer to sign again.
- **429:** back off and follow any retry guidance; confirm limits with support.
- **Server error:** do not infer that payment failed solely from the HTTP status; inspect any receipt and reconcile.

Use integer amounts: 1 Base USDC = 1,000,000 atomic units; 1 BNB Binance-Peg USDC = 1,000,000,000,000,000,000 atomic units. Do not use floating-point arithmetic for payment amounts.

## Validation scope

The hosted documentation was checked on 9 October 2026. Public discovery and authentication rejection checks do not prove a complete paid integration. Existing on-chain settlement evidence is available below. The corresponding authenticated API responses and resource-delivery evidence have not been reviewed, so end-to-end API validation remains unconfirmed. No new payment was initiated to write this guide. This is integration guidance, not a security audit or uptime guarantee.

Report documentation issues at https://github.com/xpayllc/xpayllc/issues. Include only redacted errors and public transaction hashes; never include keys, private keys or reusable signed authorizations.

## Existing settlement evidence

- **Base:** [X Pay application with ten receipt links](https://github.com/Merit-Systems/x402scan/pull/1218). These are submitted settlement records; this documentation update did not independently revalidate all ten receipts.
- **BNB:** [Verified XPayFacilitator contract](https://bscscan.com/address/0xc27aac475a332ede1290f60a2785dcca49f40cb6#code). Its published interface uses token allowance, `settleExact` and `settleExactBatch`.
- **BNB checked receipt:** [0x0e1c56ed…35b635b](https://bscscan.com/tx/0x0e1c56edfb86dd263a222aff52b36c8dce3379728cfc905c46f97abea35b635b), successful on 9 October 2026 at 12:01:19 UTC, block 126632228. It transferred 0.002 Binance-Peg USDC from `0xddb2ca4852cc4a00896f3c6ea4f1ff219abf2a46` to the published facilitator signer `0x589a2314a2e05f45e40c4823da3ba58d421db3d8`, which submitted the transaction.

This receipt establishes a successful token settlement through that contract. It does not establish use of `/bsc/verify`, `/bsc/settle`, Permit2 or delivery of an API resource. Confirm the deployed API-to-contract mapping before choosing an integration route. Contract verification means source matching, not a security audit. No new transaction was initiated for this review.

