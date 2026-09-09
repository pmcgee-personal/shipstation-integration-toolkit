# Exception Handling

Patterns for managing exceptions and error cases in ShipStation integrations. These cover scenarios where shipments need to be corrected, recovered, or adjusted before fulfillment.

## Architecture Overview

Even in well-designed integrations, exceptions occur: incorrect addresses, wrong item quantities, weight/dimension updates, carrier changes, or split shipments. These patterns show how to detect and resolve common exception scenarios using ShipStation's update and correction APIs.

## Patterns

- [`update-shipment.md`](./update-shipment.md) — Correct shipment details before label creation (address, items, dimensions, weight)

## Key Concepts

**Pending shipment requirement:**
- Shipments can **only be updated if `shipment_status` is `"pending"`**.
- Once a label is created, the shipment becomes immutable (status changes to `processed`, `shipped`, etc.).
- Always check status before attempting updates.

**Full-object updates:**
- The `PUT /v2/shipments/{shipment_id}` endpoint requires the complete shipment payload.
- Partial updates will fail. Always fetch the current shipment first via `GET /v2/shipments/{shipment_id}`, modify fields, and send the full object back.

**Common exception scenarios:**
- Incorrect shipping address detected before label creation
- Items added or removed from the shipment
- Weight or dimensions updated after order import
- Carrier or service selection change needed
- Split shipment coordination

## Typical Flow

1. Shipment is imported and status is `pending`
2. An exception is detected (wrong address, wrong items, etc.)
3. Integrator calls `GET /v2/shipments/{shipment_id}` to fetch current state
4. Integrator modifies the problematic fields
5. Integrator calls `PUT /v2/shipments/{shipment_id}` with the complete updated payload
6. ShipStation confirms the update
7. Shipment remains in `pending` state, ready for label creation
8. If the exception is discovered after label creation, the shipment cannot be updated — a cancellation and new shipment creation may be required

## Rate Limit Impact

Exception handling is event-driven:
- GET shipment: 1 call per exception detected
- PUT update: 1 call per resolution

Low overhead; scales with exception frequency.

## Related Patterns

All integration patterns may encounter exceptions that require shipment updates:
- [Order Lifecycle](../order%20lifecycle/)
- [ERP Integration](../erp/)
- [WMS Integration](../wms/)
- [Inventory Management](../inventory/)

## References

- Official docs: [GET /v2/shipments/{shipment_id}](https://docs.shipstation.com/apis/openapi/shipments/get_shipment_by_id)
- Official docs: [PUT /v2/shipments/{shipment_id}](https://docs.shipstation.com/apis/openapi/shipments/update_shipment)
- Official docs: [Shipment Statuses](https://docs.shipstation.com/apis/openapi/shipments)

## Notes

- Check shipment status before attempting updates. Only `pending` shipments are mutable.
- Always fetch the current shipment state before updating. Never rely on cached data.
- The full object is required in updates. Omitting fields may cause data loss.
- Common update scenarios include:
  - Fixing incorrect addresses before label creation
  - Adding or removing items
  - Correcting weight/dimensions
  - Changing carrier or service selection
  - Splitting a shipment into multiple shipments
- If an exception is discovered after a label is created, the shipment cannot be updated. Consider cancellation (if supported by the carrier) and creating a new shipment.
- Document all exceptions for monitoring and process improvement. Exceptions may indicate upstream data quality issues.
