# WuloPay API specification

OpenAPI 3.1 description of the WuloPay **Hosted Checkout API**: create a checkout session for an order, send the
shopper to WuloPay's secure payment page, and get told by a signed webhook when it is paid.

- Spec: [`openapi.yaml`](openapi.yaml) (view it in any OpenAPI tool, e.g. `npx @redocly/cli preview-docs openapi.yaml`)
- Libraries and plugins built on it: [wulopay-php](https://github.com/WuloPay/wulopay-php),
  [wulopay-woocommerce](https://github.com/WuloPay/wulopay-woocommerce)

## How it works

```
Your server                          WuloPay                                   Shopper
POST /Authorization/Token/ApiToken -> token (cache it, 60 minutes)
POST /Checkout/Sessions            -> { id, url }      redirect shopper to url ->  pays on WuloPay's page
                                                                                    (card + 3-D Secure)
webhook checkout.session.completed <- signed POST to your endpoint
GET  /Checkout/Sessions/{id}       -> status "paid"     <- shopper returns to returnUrl?session_id=...
```

1. Create an API key in the WuloPay merchant portal (Product Configuration, **Hosted Checkout**). Keep the key and
   secret on your server only: never in a web page or mobile app.
2. Add a webhook endpoint in the same screen and store its signing secret.
3. For each order, create a session and redirect the shopper to `url`.
4. Mark the order paid when the `checkout.session.completed` webhook arrives (verify the signature) or when
   `GET /Checkout/Sessions/{id}` says `paid`. The redirect back to your site is not proof of payment.

## Webhook signatures

Every webhook carries `WuloPay-Signature: t=<unix seconds>,v1=<hex>`. Compute HMAC-SHA256 of `"<t>.<raw body>"`
with your endpoint's signing secret, compare it with `v1` using a constant-time comparison, and reject `t` older
than 5 minutes. Deliveries are retried for about 3 days, so ignore an event `id` you have already handled.

```php
$expected = hash_hmac('sha256', $t . '.' . $rawBody, $signingSecret);
$valid = hash_equals($expected, $v1) && abs(time() - (int) $t) <= 300;
```

## Sandbox

Base URL `https://sandbox.wulopay.com/api/v1`. No real money moves. 3-D Secure test cards:

| Card | Result |
|---|---|
| 4000 0000 0000 2701 (Visa) | Paid, no challenge |
| 4000 0000 0000 2503 (Visa) | Bank challenge: enter code `1234` |
| 5200 0000 0000 2235 (Mastercard) | Paid, no challenge |
| 4000 0000 0000 2925 (Visa) | The bank can't verify the card |

Any future expiry date and any 3-digit security code. An amount of R60.00 is declined.

## Support

Questions: contact WuloPay. Security problems: see [SECURITY.md](SECURITY.md).
