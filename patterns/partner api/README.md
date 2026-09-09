# Partner API

Onboard ShipStation accounts at scale via the Partner API: create accounts, define warehouses, connect carriers and provision multi-tenant shipping integrations.

This folder covers **programmatic account provisioning and carrier connection flows** for ShipStation partners. Use these patterns when building shipping integrations for multiple end-customers.

## Who This Is For

**ShipStation technology partners** embedding shipping capabilities into platforms, marketplaces or OEM solutions. You use the Partner API to:

- Create and manage customer accounts on behalf of shippers
- Set up warehouses (ship-from locations) for each account
- Connect carriers (BYOA, Carrier Portal or Elements)
- Fetch carrier metadata for downstream rate, label and tracking operations

## Types of Integrations

### Marketplaces

Sellers sign up on your platform. You create a ShipStation sub-account for each seller, manage their warehouse locations and let them use ShipStation API carriers. You consume the Partner API to provision accounts, retrieve carrier metadata, then use the standard Shipments/Tracking/Rates APIs downstream.

**Key pattern:** Carrier Portal (seller sets up ShipStation API wallet)

### SaaS/Commerce Platforms

**Examples:** Inventory management, OMS, fulfillment software, print-on-demand integrators

Your customers use your platform for order management, fulfillment, or production. Shipping is embedded as one workflow. You provision a ShipStation account per customer, own the carrier setup UX and surface rate shopping, label printing, and tracking within your product.

**Key pattern:** Either BYOA (customers provide carrier creds) or Carrier Portal (customers set up wallets + carriers via ephemeral token redirect).

### Technology Partners / OEM

**Examples:** Proprietary logistics software, warehouse automation, custom fulfillment platforms

You're embedding ShipStation shipping without exposing the ShipStation brand or UI. You manage accounts and carriers programmatically, fetch metadata and build custom shipping workflows. Customers see your product only.

**Key pattern:** BYOA Carrier Onboarding + custom rate/label/tracking UX built on Partner API responses.

## Integration Approaches

### Custom API Integration

Full programmatic control over account provisioning, warehouse setup and carrier connection.

**Covered in this folder:**

- [BYOA Carrier Onboarding](./byoa-carrier-onboarding.md) — Customer brings existing carrier account
- [Carrier Portal Onboarding](./carrier-portal-onboarding.md) — Customer enables wallet and selects carriers via ShipStation dashboard (ephemeral token redirect)

**Next steps:** Use standard ShipStation APIs for rates, labels and tracking.

### ShipStation Elements

Pre-built, embeddable UI components for shipping operations: label creation, rate shopping, tracking, carrier configuration, etc.

**Use Elements when** you want UI without building it yourself. Elements handles the ShipStation dashboard integration; you embed the component and handle the rest of your platform.

**Not covered in this repo.** See [official Elements documentation](https://docs.shipstation.com/apis/shipengine/docs/elements/elements-guide) for setup and component reference.

**Note:** You can combine approaches — use Partner API for account setup, Elements for customer-facing shipping UI.

## Key Concepts

### On-Behalf-Of Header

All partner operations require the `On-Behalf-Of: <accountId>` header. This tells ShipStation which account the request applies to, enabling multi-tenant isolation.

### Account → Warehouse → Carrier Chain

1. Create an account via `POST /accounts`
2. Create one or more warehouses (ship-from locations) for that account via `POST /warehouses`
3. Connect carriers via `POST /carriers/connect` (BYOA) or send the account to Carrier Portal (ephemeral token)
4. Fetch available carriers via `GET /carriers` to retrieve metadata (carrier_code, service_code, package_code)

### Carrier Metadata

After a carrier is connected, you must store its metadata locally — carrier_code, service_code, package_code, carrier_id. Use these in downstream rate and label requests.

## Workflows at a Glance

| Scenario                                                                           | Flow                                                                                                       | Link                                                        |
| ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| Customer provides existing carrier credentials (UPS, FedEx, USPS account)          | You collect creds in your UI, POST to `/carriers/connect`, handle OAuth flows if needed                    | [BYOA Carrier Onboarding](./byoa-carrier-onboarding.md)     |
| Customer has no carrier account yet, wants to fund shipping via ShipStation wallet | Redirect to ShipStation dashboard via ephemeral token; customer sets up wallet and connects carriers there | [Carrier Portal Onboarding](./carrier-portal-onboarding.md) |

## Quick Decision Tree

**Use BYOA if:**

- Customers already have carrier accounts
- You want full control of the setup UX
- You're migrating existing carrier relationships

**Use Carrier Portal if:**

- Customers don't have carrier accounts yet
- You want to offload wallet/carrier setup to ShipStation
- You prefer a simple ephemeral token redirect vs. managing carrier authentication flows

**Use Elements if:**

- You want pre-built shipping UI without building rate shopping, label printing, or tracking surfaces
- You don't mind your customers seeing ShipStation branding
- You'd rather embed than build custom

## Documentation & References

### This Folder

- [BYOA Carrier Onboarding](./byoa-carrier-onboarding.md)
- [Carrier Portal Onboarding](./carrier-portal-onboarding.md)

### Official ShipStation Docs

- [Partner Integration Guide](https://docs.shipstation.com/apis/shipengine/docs/partners/partner-integration-guide) — Authoritative specs, authentication, error codes
- [Carrier Connect Guide](https://docs.shipstation.com/apis/shipengine/docs/carriers/connect) — Carrier-specific auth requirements (OAuth, API keys, etc.)
- [Direct Login (Ephemeral Token)](https://docs.shipstation.com/apis/shipengine/docs/partners/direct-login) — Portal redirect setup
- [ShipStation Elements](https://docs.shipstation.com/apis/shipengine/docs/elements/elements-guide) — Embeddable UI components
- [API Reference](https://docs.shipstation.com/apis/shipengine/docs/reference/introduction) — Complete endpoint specs

## Notes

- **Authentication varies by carrier.** UPS uses OAuth; others use API keys or account credentials. The [Carrier Connect Guide](https://docs.shipstation.com/apis/shipengine/docs/carriers/connect) breaks down each carrier's auth method.
- **Post-connection config may be required.** Some carriers (UPS negotiated rates, FedEx signature images) need additional setup after connection. Check carrier-specific docs and expose these options in your integrator UI if applicable.
- **Warehouse represents a ship-from location.** Create one per fulfillment center, distribution hub or distinct location your customer operates.
- **This repo captures happy-path flows.** Edge cases, error handling, and retry strategies are in the official docs.
