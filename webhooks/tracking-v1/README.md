# Tracking Webhook v1 (`API_TRACK`)

## Objective

To hold real `API_TRACK` webhook bodies — the v1 tracking webhook — across
UPS, USPS, FedEx and DHL eCommerce, so a receiver can be built and tested
without waiting on a live shipment.

The v2 tracking webhook (`TRACK_EVENT_V2`) is in
[`../tracking-v2/`](../tracking-v2/README.md). The two carry the same kind of
information in similar but not identical envelopes; the differences are in
[v1 versus v2](#v1-versus-v2) below. Don't copy a fixture from one folder into
the other.

## Which one applies to you

`API_TRACK` is the tracking webhook for **ShipStation API** accounts, and
`TRACK_EVENT_V2` is the one for **ShipStation** accounts. An integrator
typically has one kind of account, not both, so you will normally need only
one of the two folders. If your account is a ShipStation account, use
[`../tracking-v2/`](../tracking-v2/README.md) instead.

## Like v2, the data is inline

An `API_TRACK` body carries the tracking data itself in `data`. There is
nothing to fetch and no API key is needed to read it. The body does still
carry a `resource_url`:

```
https://api.shipengine.com/v1/tracking?carrier_code=ups&tracking_number=1Z################
```

It is built from the carrier code and tracking number, not from a label ID
(v2 uses `/v2/labels/{label_id}/track`), and it is on the `api.shipengine.com`
host. Everything is already in `data`, so switch on `resource_type` first and
don't pattern-match on `resource_url`. See [`../README.md`](../README.md) for
the pointer pattern the other webhooks use.

## The body

```json
{
  "_captured_at": "2026-10-14",
  "resource_url": "https://api.shipengine.com/v1/tracking?carrier_code=ups&tracking_number=1Z################",
  "resource_type": "API_TRACK",
  "data": {
    "label_url": null,
    "tracking_number": "1Z################",
    "status_code": "DE",
    "status_detail_code": "DELIVERED",
    "status_description": "Delivered",
    "events": ["…newest first"]
  }
}
```

`data` has 14 keys and every event has 19, in the same order, in all 16
fixtures. `_captured_at` is a repo convention, not something ShipStation
sends — see the [root README](../../README.md).

The top-level status fields describe the newest scan. `status_code`,
`status_detail_code`, `status_detail_description` and `carrier_detail_code`
match it in every fixture, and top-level `carrier_status_code` equals the
newest event's `event_code`. Top-level `carrier_status_description` is the
newest scan's `description`, except for USPS, where it is a full sentence
with a time and place. Top-level `status_description` is always populated,
even for UPS, whose events carry `null`.

## Carriers

| Folder                               | Carrier       | `carrier_code` in `resource_url` | Fixtures | `label_url`  |
| ------------------------------------ | ------------- | -------------------------------- | -------- | ------------ |
| [`ups/`](./ups/)                     | UPS           | `ups`                            | 4        | `null`       |
| [`usps/`](./usps/)                   | USPS          | `usps`                           | 4        | `null`       |
| [`fedex/`](./fedex/)                 | FedEx         | `fedex`                          | 4        | URL (masked) |
| [`dhl-ecommerce/`](./dhl-ecommerce/) | DHL eCommerce | `dhl_global_mail`                | 4        | URL (masked) |

The v1 body has no `carrier_code` or `carrier_id` field. The carrier appears
only in the `resource_url` query string, and the values differ from v2: USPS
is `usps` here and `stamps_com` in v2, and FedEx is `fedex` here and
`fedex_walleted` in v2.

Each folder has its own README covering that carrier's quirks.

## Scenarios

Every carrier has the same four scenarios:

| Fixture                           | What it shows                                                     |
| --------------------------------- | ----------------------------------------------------------------- |
| `api-track-label-created.json`    | One event: the label exists, the carrier has not taken the parcel |
| `api-track-in-transit.json`       | Mid-journey, a few scans accumulated                              |
| `api-track-out-for-delivery.json` | Out for delivery today                                            |
| `api-track-delivered.json`        | Full lifecycle to delivery                                        |

| Scenario           | UPS | USPS | FedEx | DHL eCommerce |
| ------------------ | --- | ---- | ----- | ------------- |
| Label created      | ✅  | ✅   | ✅    | ✅            |
| In transit         | ✅  | ✅   | ✅    | ✅            |
| Out for delivery   | ✅  | ✅   | ✅    | ✅            |
| Delivered          | ✅  | ✅   | ✅    | ✅            |
| Delivery exception | —   | —    | —     | —             |

There is no delivery-exception capture for any carrier, and no placeholder
file for one.

**The four fixtures for a carrier are one parcel at four points in its
journey, not four parcels.** Each fixture's events appear unchanged in the
next one, so you can diff two fixtures to see what a single new scan does to
the top-level fields. This differs from v2, where every fixture is a
different parcel. The one wording change between snapshots is on USPS: the
`Out for Delivery` scan reads `Out for Delivery, Expected Delivery Between
11:15am and 1:15pm` in the out-for-delivery fixture and plain `Out for
Delivery` in the delivered one. Don't key on `description`.

Every fixture holds the full history so far, so treat bodies as cumulative
and de-duplicate on `(tracking_number, occurred_at, event_code)` rather than
treating each POST as one new scan.

## v1 versus v2

|                                                              | v1 (`API_TRACK`)                                                  | v2 (`TRACK_EVENT_V2`)                            |
| ------------------------------------------------------------ | ----------------------------------------------------------------- | ------------------------------------------------ |
| `resource_url`                                               | `api.shipengine.com/v1/tracking?carrier_code=…&tracking_number=…` | `api.shipstation.com/v2/labels/{label_id}/track` |
| `data` keys                                                  | 14                                                                | 16                                               |
| `label_url`                                                  | **Present** (`null` for UPS and USPS, a URL for FedEx and DHL)    | Absent                                           |
| `tracking_url`, `carrier_code`, `carrier_id`                 | Absent                                                            | Present                                          |
| Event keys                                                   | 19                                                                | 20                                               |
| Event `event_description`                                    | **Present**, always equal to `description`                        | Absent                                           |
| Event `carrier_status_code` and `carrier_status_description` | Absent                                                            | Present                                          |
| Event `status_description`                                   | `null` on every UPS event, populated for the other carriers       | Populated everywhere                             |

Event-level `status_code` and `status_detail_code` are present in both.
Field names are otherwise shared.

## Carrier differences that will break shared code

The envelope is identical across carriers. The **values** are not.

| Behaviour                                      | UPS                       | USPS                                                            | FedEx                     | DHL eCommerce                     |
| ---------------------------------------------- | ------------------------- | --------------------------------------------------------------- | ------------------------- | --------------------------------- |
| `status_detail_code`                           | Populated                 | Populated                                                       | Populated                 | **`null` everywhere**             |
| `carrier_detail_code`                          | Populated (`MP`, `OR`, …) | `null`                                                          | `null`                    | `null`                            |
| `carrier_status_code` style                    | Letter (`M`, `I`, `D`)    | Scan code (`GX`, `TL`, `01`)                                    | Two letters (`OC`, `AR`)  | Number as a string (`238`, `598`) |
| Event `status_description`                     | `null`                    | Populated                                                       | Populated                 | Populated                         |
| `signer`                                       | `""`                      | `null`                                                          | `null`                    | `null`                            |
| `postal_code` when absent                      | `""`                      | `null` on scans with no location                                | `null` on **every** scan  | Populated on every scan           |
| `city_locality` / `state_province`             | `""` on the label scan    | `null` on no-location scans; facility name in city on two scans | `null` on the label scan  | **`null` on every scan**          |
| `country_code`                                 | Always `US`               | Always `US`                                                     | Always `US`               | Always `US`                       |
| `carrier_occurred_at`                          | No suffix, carrier-local  | `-07:00` offset, same instant as `occurred_at`                  | `Z`, equals `occurred_at` | `Z`, equals `occurred_at`         |
| `actual_delivery_date` vs the `Delivered` scan | Match exactly             | Naive, **7h earlier**                                           | Naive, **14h earlier**    | Naive, **7h earlier**             |
| Events sorted by `occurred_at`                 | Yes                       | Yes                                                             | Yes                       | **No** — one pair is swapped      |

`status_detail_code` is the field to branch on for UPS, USPS and FedEx. It
does not map one event to one state: USPS reports `HUB_SCAN_OUT` on an
`Arrived at USPS Facility` scan, FedEx uses `AR` for two differently worded
scans, and UPS uses `Processing at UPS Facility` for two different detail
codes. DHL eCommerce has no `status_detail_code`, so a
receiver has to use `status_code` plus the numeric `event_code`.

`status_code` does not only move forward. FedEx's `PICKED_UP` scan (`AC`) is
timestamped before its label-created scan (`NY`), so oldest to newest the
sequence is `AC`, `NY`, `IT`.

Location fields are not uniformly present. Test each field for absence, and
treat `""` and `null` as the same case.

## Timestamps

`occurred_at` is UTC and is the one to parse. `carrier_occurred_at` is
encoded three ways across this folder:

| Encoding                    | Carrier    | Actually UTC?                                                                                                                                                                          |
| --------------------------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `2026-10-14T16:25:47`       | UPS        | **No** — naive carrier-local time. 4h behind `occurred_at` in these fixtures, which only cover Connecticut, New Jersey and New York                                                    |
| `2026-10-01T12:24:49-07:00` | USPS       | Yes — a real offset, and the same instant as `occurred_at`. But the offset is `-07:00` on **every** scan, including Louisiana and Arizona ones, so it is not the scan's own local time |
| `2026-10-14T19:46:29Z`      | FedEx, DHL | Identical to `occurred_at`. For DHL, one pair of scans suggests `occurred_at` itself is not always a true UTC instant — see [`dhl-ecommerce/`](./dhl-ecommerce/README.md#event-order)  |

The date fields at the top level are looser. `ship_date`,
`estimated_delivery_date` and `actual_delivery_date` have no consistent
suffix:

- **UPS:** `ship_date` and `estimated_delivery_date` are the label date at
  midnight with no suffix. `actual_delivery_date` is `Z`-suffixed and equals
  the `Delivered` scan's `occurred_at`.
- **USPS, FedEx, DHL:** `ship_date` is the first physical scan expressed in
  UTC−7 with no suffix (for example USPS `13:27:58` for a scan at
  `20:27:58Z`). `actual_delivery_date` is also naive and does not match the
  `Delivered` scan, as the table above shows.

One payload can therefore mix suffixed and unsuffixed dates. Parse
`occurred_at`, use `carrier_occurred_at` for display only, and sort by
`occurred_at` rather than trusting array position. See
[`../tracking-v2/README.md`](../tracking-v2/README.md#carrier_occurred_at-is-not-utc)
for the same warning on v2.

## Ordering and duplicates

Events are newest first and are sorted in 13 of the 16 fixtures. The three
DHL eCommerce fixtures that contain the pair (in-transit, out-for-delivery
and delivered) each have one swapped pair: the scan at `2026-10-09T06:34:37Z`
is listed before the one at `06:51:25Z`. There are no duplicate scans in any
v1 fixture, unlike the USPS v2 fixtures.

## Sanitization

Tracking numbers are fully masked with their original length preserved: `#`
characters for UPS (18), FedEx (12) and DHL eCommerce (30), and `X`
characters for USPS (22). The `resource_url` matches the masked number. The
`se-` ID in the FedEx and DHL `label_url` is masked as `se-#########`.

Coordinates are rounded to 2 decimal places. Raw bodies arrive at 4
decimals, and a `Delivered` scan can be at delivery-address precision. The
delivery ZIP is kept as received.

No names, addresses or phone numbers appear. `signer` is `null` or `""` and
`proof_of_delivery_url` is `null` throughout. See
[`../tracking-v2/README.md`](../tracking-v2/README.md#pii-in-tracking-payloads).

## Related

The same data is reachable by polling — see
[`api/tracking/usps/`](../../api/tracking/usps/README.md). Webhook and API
shapes for the same resource are similar but not guaranteed identical.
