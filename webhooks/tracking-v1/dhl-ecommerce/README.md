# DHL eCommerce Tracking Webhook v1 (`API_TRACK`)

## Objective

To hold real `API_TRACK` webhook bodies for a DHL eCommerce parcel at each
stage of the lifecycle, so a receiver can be built and tested without
waiting on a live shipment.

Tracking data is delivered **inline in the POST body** — there is no
follow-up call to make. See [`../README.md`](../README.md) for why, and for
the fields shared by all four carriers. There are no v2 DHL eCommerce
fixtures in this repo.

All fixtures here have `carrier_code=dhl_global_mail` in the `resource_url`.
The body itself carries no `carrier_code` or `carrier_id`.

## Scenarios

| Fixture                                                                | `status_code` | `carrier_status_code` | Events | What it shows                                                                              |
| ---------------------------------------------------------------------- | ------------- | --------------------- | ------ | ------------------------------------------------------------------------------------------ |
| [`api-track-label-created.json`](./api-track-label-created.json)       | `AC`          | `238`                 | 1      | `LABEL CREATED`. `ship_date` and ETA are `null`.                                           |
| [`api-track-in-transit.json`](./api-track-in-transit.json)             | `IT`          | `540`                 | 9      | Through the DHL network and into a USPS sort facility. One pair of events is out of order. |
| [`api-track-out-for-delivery.json`](./api-track-out-for-delivery.json) | `IT`          | `598`                 | 11     | `OUT FOR DELIVERY`.                                                                        |
| [`api-track-delivered.json`](./api-track-delivered.json)               | `DE`          | `600`                 | 12     | `DELIVERED`. ETA is still populated and is later than the delivery.                        |

The four fixtures are **one parcel at four points in its journey**. Each
fixture's events appear unchanged in the next. The events array is
cumulative.

## No `status_detail_code`

DHL eCommerce is the one carrier here with no detail code. In every
fixture, `status_detail_code` and `status_detail_description` are `null`, at
the top level and on every event, and so is `carrier_detail_code`. The
branching fields are `status_code` (`AC`, `IT`, `DE`) and the numeric
`event_code`, which equals the top-level `carrier_status_code` for the newest
scan.

| `status_code` | `event_code` | `description`                                                                                  |
| ------------- | ------------ | ---------------------------------------------------------------------------------------------- |
| `AC`          | `238`        | `LABEL CREATED`                                                                                |
| `IT`          | `132`        | `EN ROUTE TO DHL ECOMMERCE OR AWAITING PROCESSING`                                             |
| `IT`          | `200`        | `PACKAGE RECEIVED AT DHL ECOMMERCE DISTRIBUTION CENTER`                                        |
| `IT`          | `220`        | `PROCESSED`                                                                                    |
| `IT`          | `300`        | `DEPARTURE ORIGIN DHL ECOMMERCE FACILITY`                                                      |
| `IT`          | `350`        | `ARRIVAL DESTINATION DHL ECOMMERCE FACILITY`                                                   |
| `IT`          | `400`        | `TENDERED TO DELIVERY SERVICE PROVIDER, ALLOW 1-3 DAYS FOR UPDATES FOR PACKAGES WITHIN THE US` |
| `IT`          | `540`        | `PROCESSED THROUGH USPS SORT FACILITY` (seen twice)                                            |
| `IT`          | `520`        | `ARRIVAL AT POST OFFICE`                                                                       |
| `IT`          | `598`        | `OUT FOR DELIVERY`                                                                             |
| `DE`          | `600`        | `DELIVERED`                                                                                    |

The codes are in chronological order for the delivered fixture, not array
order (see [Event order](#event-order)). The `400` scan reads as a hand-off to
a delivery service provider, and the later scans mention a USPS sort facility
and a post office. A receiver can't tell _out for delivery_ from _in transit_
without checking for `598`, because `status_code` is `IT` for both.

## Location fields

DHL eCommerce reports a **ZIP and coordinates, with no city or state**:

- `city_locality` and `state_province` are `null` on every scan.
- `postal_code` is populated on every scan, including the label scan. Four
  different ZIPs appear in the delivered fixture.
- `latitude` and `longitude` are populated on every scan.
- `country_code` is `US` on every scan.
- `signer`, `company_name` and `proof_of_delivery_url` are `null`
  throughout.

This is the opposite of FedEx, which has a city and state but no ZIP. A
receiver that builds a display location from `city_locality` and
`state_province` gets nothing for DHL.

## Event order

Events are meant to be newest first, but each of the three fixtures that
contain it has one swapped pair: the scan at `2026-10-09T06:34:37Z` is
listed before the scan at `2026-10-09T06:51:25Z`. It is the same pair in the
in-transit, out-for-delivery and delivered fixtures. Sort by `occurred_at`
rather than trusting array position.

The pair is `DEPARTURE ORIGIN DHL ECOMMERCE FACILITY` (`300`, ZIP `75062`)
and `ARRIVAL DESTINATION DHL ECOMMERCE FACILITY` (`350`, ZIP `98390`).
Sorted by `occurred_at` they read as the right sequence, but they are 17
minutes apart, between a ZIP in Texas and a ZIP in Washington State, so at
least one of the two timestamps is not a true UTC instant. Don't compute
transit durations from DHL scan times.

## Timestamps and dates

`carrier_occurred_at` is `Z`-suffixed and identical to `occurred_at` on
every scan.

The top-level dates are naive (no suffix):

- `ship_date` (`2026-10-07T11:52:48`) is the first scan after the label,
  expressed in UTC−7. That scan is `2026-10-07T18:52:48Z`. It is `null` on
  the label-created fixture.
- `estimated_delivery_date` is `2026-10-16T17:00:00` on every fixture after
  the label, including the delivered one. The parcel arrived on
  `2026-10-14`, two days earlier, and the ETA was not cleared.
- `actual_delivery_date` is `2026-10-14T12:28:00`. The `Delivered` scan is
  `2026-10-14T19:28:00Z`, so the date is 7 hours earlier than the scan if
  read as UTC.

## Sanitization

The tracking number is masked as 30 `#` characters (length preserved) and
the `resource_url` matches. `label_url` is a URL whose `se-` ID is masked
(`se-#########`). There is no `tracking_url` in v1.

Coordinates are rounded to 2 decimal places. ZIP codes on scans are kept as
received. No names, addresses or phone numbers appear. See
[`../README.md`](../README.md#sanitization).
