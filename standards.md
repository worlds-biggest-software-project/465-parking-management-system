# Standards & API Reference

> Project: Parking Management System · Generated: 2026-05-07

---

## Industry Standards & Specifications

### ISO Standards

**ISO TS 5206-1:2023 — Intelligent Transport Systems: Parking — Part 1: Core Data Model**
- URL: https://www.iso.org/standard/80997.html
- Adopted by ISO TC204 (Intelligent Transport Systems). Defines terms, concepts, and a Model Driven Architecture data model for both on-street and off-street parking. Covers Place, Right, Rate, Quote, Occupancy, Session, and Observation entities. Based on the APDS v3 specification and forms the canonical international data model for parking data exchange.

**ISO 16787:2016 — Intelligent Transport Systems: Assisted Parking System (APS) — Performance Requirements and Test Procedures**
- URL: https://www.iso.org/standard/63626.html
- Specifies performance requirements and test procedures for Assisted Parking Systems (APS) in vehicles — the automated sensor-based parking assist systems. Relevant for integrating smart vehicle data with facility occupancy systems.

**ISO 14823 — Graphic Data Dictionary / Graphic Vocabulary for Road Signs and Symbols (ITS)**
- URL: https://www.iso.org/standard/73182.html
- Specifies graphic symbols used in ITS applications including parking guidance and information, applicable to digital signage systems integrated with parking availability data.

---

### W3C & IETF Standards

**RFC 9110 — HTTP Semantics (IETF, 2022)**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- The foundational specification for HTTP request/response semantics. All REST APIs in parking management platforms must conform to RFC 9110 for correct use of HTTP methods (GET, POST, PUT, PATCH, DELETE), status codes, and headers.

**RFC 6749 — The OAuth 2.0 Authorization Framework (IETF, 2012)**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The standard authorisation framework used by virtually all parking management APIs for delegated access. Required for any API that allows third-party apps (navigation apps, fleet systems, city portals) to access parking data on behalf of operators or drivers.

**RFC 7519 — JSON Web Token (JWT) (IETF, 2015)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Compact claims representation used extensively for API authentication tokens in SaaS platforms including parking management systems. Used alongside OAuth 2.0 for bearer token issuance.

**RFC 8288 — Web Linking (IETF, 2017)**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Specifies link relations used in REST API hypermedia responses. Relevant for building HATEOAS-compliant parking APIs that expose navigable resource relationships.

**W3C Geolocation API Specification**
- URL: https://www.w3.org/TR/geolocation/
- Browser Geolocation API used in web-based parking portals and mobile apps to locate the user and display nearby available spaces, zones, and facilities.

---

### Data Model & API Specifications

**APDS (Alliance for Parking Data Standards) Specification v3.0**
- URL: https://allianceforparkingdatastandards.org/specifications/
- The international consensus parking data specification maintained by the APDS (a joint initiative of the British Parking Association, European Parking Association, International Parking & Mobility Institute, and National Parking Association). Version 3.0 covers Place, Right, Rate, Quote, Occupancy, Session, and Observation. Freely available to registered members. The basis for ISO TS 5206-1. Any interoperable parking platform should implement this model for data exchange.

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for describing REST APIs in a machine-readable format. Parking management platforms exposing developer APIs should publish an OpenAPI 3.1 document. Supports webhook definitions (added in OAS 3.1) relevant to occupancy events, session updates, and citation events.

**GeoJSON (RFC 7946, IETF, 2016)**
- URL: https://www.rfc-editor.org/rfc/rfc7946
- The standard format for encoding geographic data structures (points, polygons, feature collections). Essential for representing parking facility locations, zone boundaries, space layouts, and LPR patrol routes in APIs and maps.

**CEN/TS 16157-6:2022 — DATEX II Parking Data Exchange Standard**
- URL: https://standards.iteh.ai/catalog/standards/cen/7dbd0d6b-2f7d-4cc1-87ae-3b7d0ec80baa/cen-ts-16157-6-2022
- European standard for exchanging parking data between traffic management centres and operators. Defines static content (parking area descriptions, spot types, rates) and dynamic content (real-time occupancy and measurement). Used extensively by European municipalities and national access points (NAPs). Relevant for systems targeting European deployments.

**NTCIP 2304 — National Transportation Communications for ITS Protocol (NTCIP)**
- URL: https://www.ntcip.org/
- US standard developed by AASHTO/ITE/NEMA for centre-to-centre and centre-to-field ITS communication. Relevant for integration with US traffic management centres and parking guidance information systems. Supports DATEX ASN.1 at the application layer.

**JSON:API Specification**
- URL: https://jsonapi.org/
- A widely adopted convention for structuring JSON API responses including pagination, filtering, sorting, and sparse fieldsets. Suitable as the response format for a RESTful parking management API to reduce inconsistency between endpoints.

---

### Security & Authentication Standards

**PCI DSS v4.0 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/
- Mandatory for any parking platform that processes, stores, or transmits cardholder data. Covers six objectives and 12 core requirements. Parking operators are classified as merchants under PCI DSS; the compliance level (Level 1–4) depends on annual transaction volume. Point-to-Point Encryption (P2PE) is strongly recommended to reduce the scope of PCI DSS assessment.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer built on top of OAuth 2.0. Used for Single Sign-On (SSO) integration with university, municipal, or corporate identity providers (Active Directory, LDAP, Google Workspace, etc.), enabling permit eligibility verification based on role or affiliation.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Best-practice security guidance for REST API implementations. A parking management API handles sensitive PII (vehicle registrations, payment data, location history) and citation records, making OWASP API Top 10 compliance essential for secure API design.

**NIST SP 800-53 — Security and Privacy Controls for Information Systems**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- The US federal framework for security controls. Relevant for parking platforms deployed by municipal governments or public universities subject to federal compliance requirements.

**SOC 2 Type II (AICPA)**
- URL: https://www.aicpa.org/resources/article/soc-2
- Service Organisation Control audit framework. Municipal and enterprise buyers increasingly require SOC 2 Type II certification from parking SaaS vendors, particularly for systems handling payment card data and citizen PII.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr-info.eu/
- European data protection regulation. Parking systems collecting licence plate data, location history, and payment information from EU residents must comply with GDPR requirements for data minimisation, purpose limitation, subject access rights, and data breach notification. LPR data is explicitly considered personal data under GDPR guidance.

---

### EV Charging & Accessibility Standards

**OCPI 2.3 — Open Charge Point Interface**
- URL: https://evroaming.org/ocpi-protocol/
- Open protocol for EV charging roaming and data exchange, published under Creative Commons Attribution-NoDerivatives 4.0. Version 2.3 (released February 2025) adds a Parking module for CPOs to report parking spot counts and properties, directly relevant to integrating EV charging bay management with parking management systems. Required for EU AFIR (Alternative Fuels Infrastructure Regulation) compliance.

**OCPP 2.0.1 — Open Charge Point Protocol**
- URL: https://www.openchargealliance.org/protocols/ocpp-201/
- Protocol for communication between EV charging station hardware and a management backend. If a parking platform manages EV charging stations, OCPP 2.0.1 is the standard for controlling charger sessions and reporting energy data.

**ADA Standards for Accessible Design (US DOJ / US Access Board)**
- URL: https://www.access-board.gov/ada/
- Legal requirements for accessible parking spaces (quantity, dimensions, signage) and, per proposed 2024 rulemaking, accessible EV charging stations. Software interfaces for payment kiosks and self-service portals must also meet WCAG 2.1 AA or higher for screen reader and motor accessibility.

**WCAG 2.1 / 2.2 — Web Content Accessibility Guidelines (W3C)**
- URL: https://www.w3.org/TR/WCAG22/
- Accessibility guidelines for web and mobile applications. All driver-facing portals and operator dashboards should target WCAG 2.1 AA as a minimum to comply with ADA/Section 508 requirements in the US and the European Accessibility Act (EAA) in the EU.

---

## Similar Products — Developer Documentation & APIs

### Passport

- **Description:** End-to-end municipal parking compliance platform covering mobile pay, digital permits, and LPR enforcement.
- **API Documentation:** Available to municipal partners via Passport integration portal; open API architecture documented at https://www.passportinc.com/blog/leverage-apis-to-create-smarter-cities-through-a-single-platform/
- **SDKs/Libraries:** Not publicly released; integration via REST API
- **Developer Guide:** https://www.passportinc.com/blog/passports-platform-integrations/
- **Standards:** REST/JSON; OpenAPI not publicly published
- **Authentication:** OAuth 2.0 for municipal and third-party integrations

---

### T2 Systems

- **Description:** Campus and municipal permit management and enforcement platform with PARCS integration.
- **API Documentation:** Partner-accessible; details at https://www.t2systems.com/
- **SDKs/Libraries:** Not publicly released
- **Developer Guide:** https://www.t2systems.com/parcs-parking-access/
- **Standards:** REST/JSON; PARCS integration via API
- **Authentication:** SSO (SAML/OIDC) for campus identity systems; API key for PARCS integration

---

### SpotHero Developer Platform

- **Description:** Consumer parking marketplace with developer API, SDK, and Web Widget for embedding parking reservations in third-party apps.
- **API Documentation:** https://developers.spothero.com/
- **SDKs/Libraries:** iOS SDK, Android SDK, JavaScript Web Widget — https://spothero.com/developers
- **Developer Guide:** https://www.dannyzagorski.com/my-work-yall/2020/1/16/developer-portal-documentation-and-tools
- **Standards:** REST/JSON; OpenAPI documented on developer portal
- **Authentication:** API key + OAuth 2.0 for partner integrations

---

### ParkMobile Developer API

- **Description:** Mobile payment platform for on-street and off-street parking with SDK for embedding payment flows in navigation and fleet apps.
- **API Documentation:** https://developer.parkmobile.io/
- **SDKs/Libraries:** iOS SDK, Android SDK, Web SDK — https://developer.parkmobile.io/
- **Developer Guide:** https://parkmobile.io/parking-providers/integrations
- **Standards:** REST/JSON; OpenAPI available on developer portal
- **Authentication:** OAuth 2.0

---

### HERE Parking APIs

- **Description:** HERE provides two distinct REST APIs: Off-Street Parking (facility locations, opening times, height restrictions, real-time availability) and On-Street Parking (street-level spot availability, occupancy, tariffs, restrictions).
- **API Documentation:**
  - Off-Street: https://developer.here.com/documentation/off-street-parking/dev_guide/topics/overview.html
  - On-Street: https://developer.here.com/documentation/on-street-parking/dev_guide/index.html
- **SDKs/Libraries:** HERE Maps SDK for iOS, Android, and JavaScript — https://developer.here.com/
- **Developer Guide:** https://www.here.com/learn/blog/integrating-here-on-street-parking-api-via-here-sdk
- **Standards:** REST/JSON; GeoJSON for spatial data
- **Authentication:** HERE API key; OAuth 2.0 for enterprise accounts

---

### Parkopedia Parking Data API

- **Description:** Global parking database API providing static and dynamic parking data (locations, prices, availability) for in-vehicle navigation and mobility apps.
- **API Documentation:** https://business.parkopedia.com/parking-data
- **SDKs/Libraries:** REST API and data feed; integration documentation at https://business.parkopedia.com/solutions/mobile-navigation
- **Developer Guide:** https://business.parkopedia.com/
- **Standards:** REST/JSON and data feed formats
- **Authentication:** API key; enterprise agreement required

---

### Parklio Parking API & SDK

- **Description:** IoT-focused parking platform offering a RESTful API and mobile SDKs for integrating smart barrier control, space sensing, and reservation management into navigation and mobility apps.
- **API Documentation:** https://parklio.com/en/parking-software/parklio-api
- **SDKs/Libraries:** Android SDK, iOS SDK — https://parklio.com/en/parking-software/parklio-api
- **Developer Guide:** https://parklio.com/en/parking-software/parklio-api
- **Standards:** REST/JSON; API responses in standard JSON format
- **Authentication:** API key

---

### EasyPark Developer Portal

- **Description:** European mobile parking payment platform providing APIs for operators and third-party integration with the EasyPark driver network.
- **API Documentation:** https://developer.easyparkgroup.com/
- **SDKs/Libraries:** REST API; SDK details at developer portal
- **Developer Guide:** https://developer.easyparkgroup.com/
- **Standards:** REST/JSON; OpenAPI documented on portal
- **Authentication:** OAuth 2.0

---

### INRIX On-Street Parking API

- **Description:** Real-time on-street parking availability, block-level occupancy, and pricing data for navigation, fleet, and smart city applications.
- **API Documentation:** https://docs.inrix.com/parking/blocks/
- **SDKs/Libraries:** REST API; INRIX SDK for connected vehicle and navigation integration
- **Developer Guide:** https://developer.inrix.com/
- **Standards:** REST/JSON; GeoJSON for spatial data
- **Authentication:** API key + OAuth 2.0

---

## Notes

- **APDS / ISO 5206 adoption lag**: While ISO TS 5206-1 was formally adopted in 2023, most commercial parking platforms do not yet expose data in a fully APDS-compliant format. Building native APDS support into a new platform would be a meaningful differentiator for enterprise and municipal buyers.
- **OCPI 2.3 as bridge**: The inclusion of a Parking module in OCPI 2.3 creates a natural convergence point between EV charging networks and parking management systems. Platforms that implement both OCPI parking objects and standard permit/enforcement APIs will be well-positioned for the EV-integrated smart facility market.
- **LPR vendor fragmentation**: There is no universal standard for LPR camera data exchange. Each major vendor (Vigilant, Genetec, Genetec AutoVu, Digital Recognition Network) has a proprietary API. An abstraction layer that normalises LPR feeds across vendors is an underserved infrastructure component.
- **GDPR and LPR**: In the EU, licence plate data collected by LPR systems is personal data. Privacy-by-design architecture (data minimisation, retention limits, purpose limitation) must be embedded from the start for EU deployment.
