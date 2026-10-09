# EnergyShare backend — production wallet/payment flow

## OPay Transfer
1. The backend creates a unique `FUND_*` funding reference.
2. The frontend displays the EnergyShare OPay receiving account and exact amount.
3. The user transfers the exact amount and uses the funding reference as narration/reference.
4. OPay notifications are received at `POST /api/wallet/webhook/opay`.
5. The backend validates the notification, reference, amount and NGN currency.
6. Wallet credit is performed only through the atomic `confirm_wallet_funding` Supabase RPC.
7. Duplicate notifications cannot credit the wallet twice.
8. If automatic OPay notification is not enabled for the merchant account, the funding remains pending until an authorized admin verifies it.

## OPay API/POS checkout
The backend also contains a documented OPay RSA adapter for merchants whose OPay account has API/POS checkout enabled.

Routes:
- `POST /api/wallet/opay/initialize`
- `GET /api/wallet/opay/verify?reference=...` or `?orderNo=...`
- `POST /api/wallet/webhook/opay`

Configure the merchant-specific OPay credentials supplied by OPay. Never invent or expose keys.

## Paystack / Flutterwave
Existing Paystack and Flutterwave accounting remains separate. Both providers credit the wallet only after server-side validation and the same atomic `confirm_wallet_funding` RPC.

Flutterwave webhook authentication uses the configured `verif-hash` secret with a timing-safe comparison. Provider `successful` is never stored as the application's wallet status; the application uses `confirmed`.

## Environment
For OPay transfer:
- `OPAY_ACCOUNT_NUMBER`
- `OPAY_ACCOUNT_NAME`
- `OPAY_BANK_NAME`
- optional `OPAY_WEBHOOK_SECRET`
- optional `OPAY_SIGNATURE_HEADER`
- optional `OPAY_SIGNATURE_ALGORITHM`

For OPay API/POS checkout:
- `OPAY_BASE_URL`
- `OPAY_CLIENT_AUTH_KEY`
- `OPAY_MERCHANT_ID`
- `OPAY_HEAD_MERCHANT_ID`
- `OPAY_MERCHANT_PRIVATE_KEY`
- `OPAY_PUBLIC_KEY`
- `OPAY_SUB_SCENE_ENUM`
- `OPAY_POS_SN`

All provider credentials and the OPay private key remain backend-only.

## VTpass production electricity / airtime / data flow

EnergyShare uses VTpass as the provider-of-record for vending. Electricity purchases always verify the destination meter with `merchant-verify` before reserving wallet funds. A persistent VTpass `request_id` is saved before `/pay`; successful transactions are finalized only when the provider reports a delivered/successful state. Pending, ambiguous and timeout cases remain `processing` and are recovered through `/requery` and the background reconciliation worker.

The airtime integration maps the frontend's legacy network names to VTpass's documented service IDs: `mtn`, `glo`, `airtel`, and `etisalat` (9mobile). Data plans are fetched from VTpass `service-variations`, and the server validates the selected variation price before debiting the wallet. 9mobile data is mapped to VTpass's documented `etisalat-data` service ID.

Apply SQL migrations 001 → 004 in order before going live.


## Airtime/Data cashback wallet (development build)

Airtime and data purchases have **₦0 EnergyShare service fee**.

Cashback rules:
- Every completed ₦1,000 of qualifying airtime/data spend in an Africa/Lagos calendar day earns ₦3.
- ₦2,000 earns ₦6, ₦3,000 earns ₦9, etc.
- Cashback is stored separately from the normal wallet balance and carries forward until used.
- Existing cashback can be applied to airtime/data purchases. Cashback is consumed first; the remaining amount is taken from the wallet.
- Cashback earned by a purchase becomes available only after VTpass confirms that purchase as completed.
- Failed/pending provider transactions do not earn cashback; if cashback was used on a failed transaction, it is restored.
- Electricity purchases/transfers do not participate in this cashback program.

Run `sql/009_cashback_wallet_redesign.sql` after the existing migrations (including `008_daily_service_cashback.sql`) in Supabase before testing the new cashback balance.


### v4.8.1 provider capability registry
- Added provider-specific meter capability rule storage (`022_provider_meter_capability_rules.sql`).
- Cached meter intelligence no longer infers remote-vending support from a configured provider.
- Electricity token routing uses the integration provider that supplied the meter verification evidence; VTPass remains electricity fallback only.


## v4.10.1 production hardening
See `PRODUCTION-GO-LIVE-CHECKLIST.md` and run `npm run audit:production` before deployment.
# EnergyShare-backend
