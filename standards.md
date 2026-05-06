# Standards & API Reference

> Project: Event Venue Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 20121:2024 — Event Sustainability Management Systems**
- **URL:** https://www.iso.org/standard/86389.html
- Published in April 2024, this voluntary international standard provides a framework to help organisations identify and manage potentially negative social, economic, and environmental impacts of events. It covers all stages of the event supply chain and applies to events of all types and sizes. Directly relevant to the sustainability tracking features increasingly demanded in venue management software.

---

### W3C & IETF Standards

**RFC 5545 — iCalendar (Internet Calendaring and Scheduling Core Object Specification)**
- **URL:** https://www.rfc-editor.org/rfc/rfc5545
- Defines the `.ics` data format for representing events, to-dos, journal entries, and free/busy information independently of any calendar service. The core standard for venue availability exports, client calendar sync, and cross-system scheduling. Key components include `VEVENT`, `VTODO`, `VFREEBUSY`, and recurrence rules. Supersedes RFC 2445.

**RFC 4791 — CalDAV (Calendaring Extensions to WebDAV)**
- **URL:** https://www.rfc-editor.org/rfc/rfc4791
- Extends WebDAV to provide a standard protocol for accessing, managing, and sharing calendar data based on iCalendar. Enables bi-directional calendar collection management (create, read, update, delete events) against a server. Extended by RFC 6638 for scheduling and RFC 7809 for time zone handling. Relevant for synchronising venue room calendars with Google Calendar, Outlook, and Apple Calendar.

**RFC 5546 — iTIP (iCalendar Transport-Independent Interoperability Protocol)**
- **URL:** https://www.rfc-editor.org/rfc/rfc5546
- Defines the protocol layer on top of iCalendar objects for group scheduling actions such as publishing, requesting, replying, cancelling, and refreshing event invitations. Enables venue systems to send booking confirmations and schedule updates interoperably across calendar clients. iMIP (RFC 6047) defines the email binding for iTIP.

**RFC 6749 & RFC 6750 — OAuth 2.0 (Authorization Framework)**
- **URL:** https://www.rfc-editor.org/rfc/rfc6749 / https://www.rfc-editor.org/rfc/rfc6750
- The de-facto standard for delegated API authorisation. Required for integrating venue management systems with Google Calendar, Microsoft 365, Stripe, and other third-party platforms. All major venue platforms (Tripleseat, Momentus, HoneyBook) use OAuth 2.0 for API authentication. Tripleseat is migrating from OAuth 1.0 to OAuth 2.0 by July 2026.

**OpenID Connect Core 1.0**
- **URL:** https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer built on top of OAuth 2.0 providing user authentication, identity tokens (JWTs), and user profile retrieval from a UserInfo endpoint. Required for SSO integration with corporate identity providers when serving enterprise venue clients and campus facilities.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- **URL:** https://spec.openapis.org/oas/v3.2.0.html / https://swagger.io/specification/
- The industry-standard machine-readable format for describing RESTful APIs using JSON or YAML. OAS 3.1 introduced full JSON Schema Draft 2020-12 compatibility. Venue management APIs (Momentus Elite, DesignMyNight Collins) expose OpenAPI documentation. New builds should publish an OpenAPI spec to enable auto-generated SDKs, documentation, and integration testing.

**AsyncAPI 3.0**
- **URL:** https://www.asyncapi.com/docs/reference/specification/v3.0.0
- The emerging standard for describing asynchronous and event-driven APIs (webhooks, WebSockets, Kafka, MQTT). Protocol-agnostic. Adoption is accelerating rapidly (downloads grew from 5M to 34M+ since 2022). Venue systems that emit webhook events (booking created, payment received, event updated) should document these with AsyncAPI to ease third-party integration.

**JSON Schema (Draft 2020-12)**
- **URL:** https://json-schema.org/
- Used within OpenAPI 3.1 for defining and validating request/response data models for venue, booking, event, contract, and invoice resources. Supports `if/then/else` conditionals and tuple validation.

---

### Security & Compliance Standards

**PCI DSS (Payment Card Industry Data Security Standard)**
- **URL:** https://www.pcisecuritystandards.org/standards/
- Mandatory for any system that processes, stores, or transmits payment card data. All event venue management platforms that handle deposits, invoices, and booking payments must comply. The standard requires network security, data encryption, access control, and annual compliance review. Delegating card handling to a certified processor such as Stripe reduces the PCI scope significantly.

**GDPR (General Data Protection Regulation)**
- **URL:** https://gdpr.eu/
- EU regulation applicable globally to any system collecting data from EU-based attendees, clients, or staff. Requires lawful basis for data collection, explicit opt-in consent for marketing, data subject rights (access, deletion, portability), and appropriate technical measures (encryption, access controls). Event venue systems that store attendee lists, contact details, and communication history must document data retention policies and provide deletion workflows.

**CCPA (California Consumer Privacy Act)**
- **URL:** https://oag.ca.gov/privacy/ccpa
- California privacy law applicable to businesses handling California residents' data. Requires transparency, opt-out mechanisms ("Do Not Sell My Data"), and clear privacy notices. Complements GDPR for North American event management deployments.

**OWASP API Security Top 10 (2023)**
- **URL:** https://owasp.org/API-Security/
- Industry reference for the most critical API security risks. Key risks for venue booking APIs include Broken Object-Level Authorization (BOLA) — where users might access another venue's bookings — Broken Authentication, Business Logic Exploitation (e.g., automating bulk booking-then-cancel), and Improper Inventory Management (deprecated API versions). Should be used as a security design checklist.

**APEX/ASTM Environmentally Sustainable Meeting Standards**
- **URL:** https://eventscouncil.org/Industry-Insights/Industry-Resources
- Nine voluntary industry standards developed by the Events Industry Council (EIC) in partnership with ASTM International (an ANSI-accredited body). Provides specifications for producing events in a sustainable manner. Also includes the APEX Event Specifications Guide (ESG), a standard template used to convey event requirements clearly to venues and suppliers. Directly models the data fields needed in BEO (Banquet Event Order) workflows.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- **URL:** https://modelcontextprotocol.io/
- An open protocol (Anthropic, 2024) enabling AI models to interact with external tools and data sources through standardised server interfaces. Relevant for the AI-native features of an event venue management system: an MCP server exposing venue availability, booking creation, BEO generation, and lead qualification tools would allow AI agents (Claude, etc.) to autonomously complete venue management workflows on behalf of users.

---

## Similar Products — Developer Documentation & APIs

### Tripleseat
- **Description:** End-to-end event and venue management SaaS for restaurants, hotels, and event spaces. Includes CRM, contracts, BEOs, invoicing, and floor plans.
- **API Documentation:** https://support.tripleseat.com/hc/en-us/sections/200821727-Tripleseat-API
- **Key Endpoints:** Events (`GET /v1/events`), Leads (`GET /v1/leads`), Bookings (`GET /v1/bookings`), Accounts, Sites, Contacts, Tasks, Menus, Webhooks
- **Authentication:** OAuth 2.0 (migrating from OAuth 1.0 — deprecated July 2026). Bearer tokens with refresh.
- **Standards:** REST/JSON. Webhooks for real-time notifications.
- **Developer Guide:** https://support.tripleseat.com/hc/en-us/articles/205162108-API-Overview

---

### Momentus Technologies (Ungerboeck / VenueOps / Priava)
- **Description:** Enterprise event management platform for convention centres, arenas, and large venues. Previously Ungerboeck and VenueOps; now unified under Momentus Technologies.
- **API Documentation:** https://help-api.venueops.com/ (Momentus Elite OpenAPI docs) · https://supportcenter.ungerboeck.com/hc/en-us/articles/115010256147-API-Overview
- **SDKs/Libraries:** C# SDK and wrapper; GitHub examples at https://github.com/UngerboeckAPI
- **Key Endpoints:** Contacts (`POST /v1/crm/contacts`), Opportunities (`POST /v1/opportunities/`), Events (`POST /v2/event/getByNumber`), Event Series (create/get)
- **Authentication:** JWT Bearer tokens generated via the Momentus SDK using API Client ID + Client Secret. HTTPS only.
- **Standards:** REST/JSON, OpenAPI specification exposed at help-api docs. Integrates with StaffSavvy for staff scheduling.
- **Developer Guide:** https://supportcenter.ungerboeck.com/hc/en-us/articles/360006782813-API-Examples

---

### DesignMyNight (Collins)
- **Description:** Venue booking and discovery platform for UK hospitality — bars, restaurants, and event spaces. Collins is its venue management and booking system.
- **API Documentation:** https://developers.designmynight.com/
- **Key APIs:** Booking API (check availability, create bookings), Venues API (venue settings and types), Offers API (eligible offers per booking), Bookings Search API, Users API, Venue Groups API
- **SDKs/Libraries:** No official SDK; REST/JSON with curl examples in docs
- **Authentication:** API key authentication
- **Standards:** REST/JSON; OpenAPI spec available at https://github.com/designmynight/collins-api-docs/blob/master/main.yaml
- **Developer Guide:** https://developers.designmynight.com/api/booking-api/

---

### Skedda
- **Description:** Self-service space booking and scheduling platform for shared workspaces, gyms, and meeting rooms.
- **API Documentation:** https://apitracker.io/a/skedda
- **Integration Capabilities:** Outgoing webhooks (booking create/update/cancel), iCal/Outlook read-only calendar feeds, Zapier and Make connectors
- **Limitation:** Intentionally blocks inbound programmatic write operations to protect core booking logic; not suitable where full API control is required.
- **Authentication:** Webhook-based; no full REST write API
- **Standards:** iCalendar for calendar feeds; webhook event format undocumented publicly
- **Developer Guide:** https://www.skedda.com/integrations/webhooks

---

### HoneyBook
- **Description:** CRM, contracts, invoicing, and client communication platform for event professionals and solo planners.
- **API Documentation:** https://apitracker.io/a/honeybook
- **Base URL:** `https://api.honeybook.com/v1/`
- **Authentication:** OAuth 2.0; Bearer token. Webhook signatures verified via HMAC-SHA256 (`x-honeybook-signature` header)
- **Webhook Events:** New client created, payment successfully paid, project stage changed
- **Standards:** REST/JSON; Zapier integration (7 triggers, 12 actions)
- **Developer Guide:** https://rollout.com/integration-guides/honey-book/api-essentials

---

### Google Calendar API (Google Workspace)
- **Description:** Google's calendar and scheduling API, supporting room and resource booking (meeting rooms, equipment, shared facilities) via the Admin SDK Directory API.
- **API Documentation:** https://developers.google.com/workspace/calendar/api/guides/overview
- **API Reference:** https://developers.google.com/workspace/calendar/api/v3/reference
- **Room/Resource API:** https://developers.google.com/admin-sdk/directory/reference/rest/v1/resources.calendars
- **SDKs/Libraries:** Official libraries for Java, JavaScript/Node.js, Python, Go, PHP, Ruby, .NET
- **Authentication:** OAuth 2.0; service accounts for server-to-server
- **Standards:** CalDAV (RFC 4791), iCalendar (RFC 5545), REST/JSON, OpenAPI
- **Integration Pattern:** Rooms booked as attendees via their resource email address; auto-accept/decline based on availability

---

### Microsoft Graph API (Microsoft 365 Bookings & Exchange Rooms)
- **Description:** Unified API for Microsoft 365 services including Exchange calendar, meeting room resources, and Microsoft Bookings for appointment scheduling.
- **API Documentation:** https://learn.microsoft.com/en-us/graph/booking-concept-overview
- **Room Resource Type:** https://learn.microsoft.com/en-us/graph/api/resources/room?view=graph-rest-1.0
- **SDKs/Libraries:** Official SDKs for .NET, JavaScript/TypeScript, Java, Python, Go, PHP, Ruby
- **Authentication:** OAuth 2.0 with Azure AD; application permissions for server-to-server room management
- **Standards:** REST/JSON, OData v4, iCalendar for calendar data, OpenAPI
- **Key Capabilities:** `findMeetingTimes`, `getSchedule` (free/busy), room list discovery, Microsoft Bookings appointment APIs

---

### Stripe Payments API
- **Description:** Payment processing API widely used in event booking SaaS for deposits, invoices, and full payment collection. Significantly reduces PCI DSS scope when used for card handling.
- **API Documentation:** https://docs.stripe.com/api
- **Webhooks:** https://docs.stripe.com/webhooks — events include `charge.succeeded`, `payment_intent.succeeded`, `invoice.paid`
- **SDKs/Libraries:** Official libraries for JavaScript/Node.js, Python, Ruby, PHP, Java, Go, .NET
- **Authentication:** API key (secret key server-side, publishable key client-side); webhook signature verification via `Stripe-Signature` header
- **Standards:** REST/JSON; PCI DSS certified processor; Stripe handles card storage (tokenisation)
- **Developer Guide:** https://docs.stripe.com/payments-api/tour

---

## Notes

**Gaps & Evolving Areas**

- **No universal venue data interchange format** exists. The event industry lacks an equivalent of HL7 FHIR (healthcare) or FIX (finance) for structured venue booking data. Most integrations rely on bespoke REST APIs or flat CSV exports. There is an opportunity for an AI-native platform to define and champion an open venue booking schema (e.g., published as a JSON Schema or OpenAPI component library).
- **BEO standardisation is absent.** The Banquet Event Order (BEO) is the core operational document in venue management, yet there is no formal data standard governing its fields, structure, or interchange format. APEX provides a template, but no machine-readable schema.
- **Caterease has no public API** as of 2026. Integration with Caterease requires proprietary connectors or screen-scraping approaches, making it a poor integration target.
- **AsyncAPI adoption in this sector is nascent.** Most venue platforms expose ad-hoc webhooks without formal AsyncAPI documentation, creating integration friction. An open-source platform with a published AsyncAPI spec would differentiate strongly.
- **ISO 20121:2024 transition.** ISO 20121:2012 validity expired in April 2026; venues seeking certification must now target the 2024 revision. Software that helps venues track and report against ISO 20121 metrics (carbon, waste, social impact) will become a compliance enabler.
