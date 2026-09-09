# Freight Management

LTL (Less Than Truckload) freight tracking and booking via ShipStation's freight API. These patterns cover querying carrier networks for freight tracking and managing freight shipments outside the standard parcel workflow.

## Architecture Overview

ShipStation's freight API allows you to track LTL shipments directly from carrier networks — whether the freight was booked in ShipStation, in an external freight system, or directly with a carrier. This is useful for inbound freight from suppliers, drop-shipments, inter-warehouse transfers, and any freight managed outside ShipStation's parcel ecosystem.

## Patterns

- [`track-freight.md`](./track-freight.md) — Query carrier networks for LTL tracking status using SCAC and Pro/BOL number
- [`pro-number-retrieval-tracking.md`](./pro-number-retrieval-tracking.md) — Retrieve Pro numbers and tracking details
- [`quote-book-print.md`](./quote-book-print.md) — Request freight quotes and create shipments

## Key Concepts

**SCAC codes:**
SCAC (Standard Carrier Alpha Code) uniquely identifies each carrier. ShipStation uses these to route freight tracking queries to the correct carrier network.

**Pro vs. BOL:**
- **Pro number** (Progressive) — carrier-assigned identifier for the shipment
- **BOL number** (Bill of Lading) — contract/invoice number

Provide either Pro or BOL along with the SCAC to query tracking.

**No ShipStation shipment required:**
Unlike parcel tracking (which ties to ShipStation shipment records), freight tracking queries the carrier directly. The shipment may have been booked outside ShipStation entirely.

## Typical Flow

1. Freight shipment is booked (in ShipStation, external freight broker, or directly with carrier)
2. You obtain the Pro number (or BOL) and carrier SCAC code
3. Query `GET /freight/tracking` with SCAC + Pro/BOL to retrieve current status
4. Monitor periodically (1-2x daily) for status updates through delivery
5. Sync tracking updates back to your ERP or supply chain visibility system

## Rate Limit Impact

Freight tracking API is **separate from the 200 req/min parcel API rate limit**. Typical queries:
- Initial tracking lookup: 1 call per shipment
- Status updates: 1 call per shipment (1-2x daily)

Low overhead; no batch limit.

## Related Patterns

- [Order Lifecycle](../order%20lifecycle/) — If freight shipments are part of fulfillment
- [Inventory Management](../inventory/) — If inbound freight affects inventory levels
- [ERP Integration](../erp/) — If the ERP manages freight logistics

## References

- Official docs: [Freight Tracking](https://docs.shipstation.com/apis/openapi/freight/get_freight_tracking)
- SCAC code lookup: [National Motor Freight Traffic Association](https://www.nmfta.org/standards/scac)
- Carrier documentation for pro/BOL retrieval varies by carrier

## Notes

- Freight API queries carrier networks in real-time; ShipStation does not store historical freight data
- Status updates are infrequent; polling 1-2x daily is sufficient
- Pro numbers are carrier-specific; always pair with the SCAC code
- Useful for supply chain visibility, inbound logistics tracking, and multi-modal shipment management
