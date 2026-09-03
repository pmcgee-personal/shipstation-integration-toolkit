# ShipStation API Adjustments for USPS

## Objective

To provide a method for programmatically retrieving United States Postal
Service (USPS) shipping adjustment reports via ShipStation API. Shipping
adjustments can occur when the carrier determines there is a discrepancy in
the original shipping details provided.

This guide outlines a two-step process to check for and download these
adjustment reports.

## Step 1: Check for available adjustment reports

To see if any new adjustment reports are available, you should query the
`/reports` endpoint once per day. It's important to note that on any given
day, there may be zero or multiple reports available.

**Endpoint**

```
GET /v1/incubator/adjustments/reports/
```

**Query parameters**

| Parameter | Type | Description | Example |
| --- | --- | --- | --- |
| `created_at_start` | string | The start of the date range (ISO 8601 format). It is recommended to query for the 24-hour period from two days prior. | `2026-03-16T00:00:00-0700` |
| `created_at_end` | string | The end of the date range (ISO 8601 format). It is recommended to query for the 24-hour period up to the previous day. | `2026-03-17T00:00:00-0700` |
| `page` | integer | The page number to retrieve. Defaults to `1`. | `1` |

**Example request**

The following request queries for reports created between March 16th, 2026,
and March 17th, 2026.

```bash
curl --request GET \
  --url 'https://api.shipengine.com/v1/incubator/adjustments/reports/?created_at_start=2026-03-16T00%3A00%3A00-0700&created_at_end=2026-03-17T00%3A00%3A00-0700&page=1' \
  --header 'API-Key: YOUR_API_KEY'
```

**Example response**

You will receive a JSON response. The `reports` array will contain an object
for each available report — see
[`list-adjustment-report-response.json`](./list-adjustment-report-response.json).

From the response, you must extract the `report_id` for each report listed in
the `reports` array.

## Step 2: Download the adjustment report

If Step 1 returns one or more reports, use the `report_id` from each to make a
`GET` request to download the contents of the report. The response will be in
CSV format.

**Endpoint**

```
GET /v1/incubator/adjustments/reports/:report_id
```

**Path parameters**

| Parameter | Type | Description |
| --- | --- | --- |
| `:report_id` | string | The `report_id` obtained from the previous step. |

**Example request**

Using a `report_id` from the previous example:

```bash
curl --request GET \
  --url https://api.shipengine.com/v1/incubator/adjustments/reports/rpt_W1XzSLtRMJzMBLmJK2vGr6 \
  --header 'API-Key: YOUR_API_KEY'
```

**Example CSV response payload**

The body of the response will be the raw CSV data — see
[`get-adjustment-report-response.json`](./get-adjustment-report-response.json).
