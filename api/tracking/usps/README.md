# ShipStation API USPS Tracking

## Objective

To provide real examples of what ShipStation API tracking
endpoint returns for USPS shipments during its lifecycle.

USPS tracking is the flow that sandbox does not reliably reproduce so there is no
practical way to see these shapes without shipping a real parcel. The
fixtures here stand in for that, one file per scenario, so a receiver can be
built and tested against every status without waiting on a live shipment.

All fixtures in this folder were captured with `carrier_code`
`stamps_com`, the carrier code USPS shipments report under.

## Retrieving tracking information

**Endpoint**

```
GET /v1/tracking
```

**Query parameters**

| Parameter         | Type   | Description                                                                  | Example                  |
| ----------------- | ------ | ---------------------------------------------------------------------------- | ------------------------ |
| `carrier_code`    | string | The carrier the tracking number belongs to. USPS shipments use `stamps_com`. | `stamps_com`             |
| `tracking_number` | string | The tracking number to look up.                                              | `9300110XXXXX3550425334` |

**Example request**

```bash
curl --request GET \
  --url 'https://api.shipengine.com/v1/tracking?carrier_code=stamps_com&tracking_number=9300110XXXXX3550425334' \
  --header 'API-Key: YOUR_API_KEY'
```

**Example response**

The response is a single tracking object: the current status at the top
level, plus an `events` array holding every scan so far, newest first — see
the scenarios below for a fixture matching the stage you need.

## Scenarios

| Fixture                                                        | `status_code` | `status_description` | What it shows                                                                                               |
| -------------------------------------------------------------- | ------------- | -------------------- | ----------------------------------------------------------------------------------------------------------- |
| [`no-scans-yet.json`](./no-scans-yet.json)                     | `NY`          | Not Yet In System    | Tracking number not recognised by USPS yet. `events` is empty and every date field is `null`.               |
| [`pre-transit.json`](./pre-transit.json)                       | `AC`          | Accepted             | Label created, parcel not yet handed over. One `Shipping Label Created` event.                              |
| [`in-transit-single-scan.json`](./in-transit-single-scan.json) | `IT`          | In Transit           | Early movement — accepted and arrived at the origin facility. Carries an `estimated_delivery_date`.         |
| [`in-transit-multi-scan.json`](./in-transit-multi-scan.json)   | `IT`          | In Transit           | Movement through a shipping partner facility before USPS possession, with several scans accumulated.        |
| [`out-for-delivery.json`](./out-for-delivery.json)             | —             | —                    | **Placeholder** — still awaiting a real capture.                                                            |
| [`delivered.json`](./delivered.json)                           | `DE`          | Delivered            | Full lifecycle, 19 events from label creation to delivery. `actual_delivery_date` is set.                   |
| [`delivery-exception.json`](./delivery-exception.json)         | `EX`          | Exception            | Carrier-network delay (`status_detail_code` `CARRIER_DELAYS`) while the parcel is still moving and on time. |
| [`returned-to-sender.json`](./returned-to-sender.json)         | `EX`          | Exception            | Undeliverable address (`No Such Number`) sending the parcel back (`status_detail_code` `RETURN_TO_SENDER`). |

## Notes

`status_code` is the coarse ShipStation status. `status_detail_code` is the
finer-grained reason, and it is what distinguishes the two `EX` fixtures
from each other — a delay and a return-to-sender share `status_code` `EX`
but mean very different things to a receiver. Don't branch on
`status_code` alone.

`carrier_status_code` and `carrier_status_description` are USPS's own
wording, passed through unchanged. They are useful for display but are not
a stable contract — prefer the ShipStation codes for logic.

An `EX` status does not necessarily mean the parcel has stopped moving; in
[`delivery-exception.json`](./delivery-exception.json) it is still in
transit and expected on time.
