# ShipStation USPS Single Payor Labels

## Objective

To document the integration requirements for creating USPS single payor
labels via ShipStation API, so the resulting labels are PCID compliant.

From an integration perspective this is like rating, labelling, or tracking
any other carrier in ShipStation API, with one key exception: when the
vendor generates the USPS label they must use `POST /v2/labels` and include
the `sender_info` object. Please note this is not documented publicly.

This solution will **not** work if the caller uses `POST /v1/rates` and then
`POST /v2/labels/rates/:rate_id`.

## Step 1: Create the label with `sender_info`

**Endpoint**

```
POST /v2/labels
```

**Body parameters**

| Parameter                             | Type   | Description                                                                                      |
| ------------------------------------- | ------ | ------------------------------------------------------------------------------------------------ |
| `shipment`                            | object | The regular shipment payload (`ship_to`, `ship_from`, `packages`, `service_code`, `carrier_id`). |
| `sender_info`                         | object | The marketplace account details for the seller. Required for PCID compliance.                    |
| `sender_info.sender_address`          | object | The seller's account address and contact details on the marketplace.                             |
| `sender_info.sender_platform_user_id` | string | The seller's unique identifier on the marketplace.                                               |

**Example request**

The following request creates a USPS Ground Advantage label with the
`sender_info` object included — see
[`create-label-single-payor-request.json`](./create-label-single-payor-request.json)
for the full body.

```bash
curl --request POST \
  --url https://api.shipstation.com/v2/labels \
  --header 'API-Key: YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data @create-label-single-payor-request.json
```

## `sender_info` vs `ship_from`

Think of `sender_info` as the seller's account information on the
marketplace (name, email, phone number, address, etc). This could be exactly
the same as the `ship_from` details, but a seller could ship from multiple
places.

For example, a seller lives in LA, so that would be their `sender_info`
(seller account address and contact details), but the item they're listing
is located at and will ship from their holiday home in TX. In that flow
`sender_info` would be the CA address and `ship_from` would be the TX
address.

## `sender_platform_user_id`

The `sender_platform_user_id` is the unique identifier for that seller on
the marketplace. This allows the seller to change their account addresses,
shipping origins, etc, while still being identifiable as the same seller.
