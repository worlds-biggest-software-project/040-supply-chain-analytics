# Standards & API Reference

> Project: Supply Chain Analytics · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO 28000:2022 — Security and Resilience — Security Management Systems**
- URL: https://www.iso.org/standard/79612.html
- Replaces ISO 28000:2007; specifies requirements for a security management system applicable to all supply chain participants. Mandatory reference for any supplier risk or cross-border logistics analytics module. Adopts the Annex SL harmonised structure, making it compatible with ISO 9001, ISO 14001, and ISO 22301.

**ISO 31000:2018 — Risk Management — Guidelines**
- URL: https://www.iso.org/standard/65694.html
- Provides principles, framework, and process for managing risk in any organisation. Directly applicable to supplier risk scoring, disruption modelling, and probabilistic inventory buffers. A supply chain analytics platform surfacing risk scores should align terminology and process steps to ISO 31000 to facilitate enterprise adoption.

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- The world's most widely adopted quality management standard. Demand forecasting accuracy (MAPE/WMAPE), supplier on-time delivery rates, and defect rates are core ISO 9001 KPIs. Any analytics platform targeting manufacturing customers will encounter ISO 9001 audit evidence requirements for supplier quality data.

**ISO 20400:2017 — Sustainable Procurement — Guidance**
- URL: https://www.iso.org/standard/63026.html
- Guidance for integrating sustainability into procurement processes. Relevant to supplier ESG scoring and emissions tracking features that underpin EU CSDDD compliance. Not a certifiable standard; provides normative vocabulary and process guidance aligned with ISO 26000.

**ISO 22301:2019 — Business Continuity Management Systems**
- URL: https://www.iso.org/standard/75106.html
- Framework for protecting against and recovering from disruptive incidents. ISO/TS 22318:2021 (supply chain continuity supplement) extends this specifically to supply chain resilience. Relevant to scenario modelling and disruption-response features aimed at enterprise risk and compliance teams.

**ISO/TS 22318:2021 — Guidelines for Supply Chain Continuity**
- URL: https://www.iso.org/standard/80536.html
- A technical specification supplementing ISO 22301 with specific guidance on supply chain continuity management. Provides the framework that enterprise supply chain planners use when designing disruption response plans; a platform implementing scenario modelling should reference this vocabulary.

---

### W3C & IETF Standards

**RFC 9110 — HTTP Semantics (2022)**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- The updated authoritative specification for HTTP request/response semantics, replacing RFC 7231. Essential foundation for REST API design across all supply chain platform integrations. Covers methods (GET, POST, PUT, PATCH, DELETE), status codes, and content negotiation.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The standard authorization framework for delegated API access. All major supply chain SaaS platforms (Blue Yonder, RELEX, Kinaxis, Flowlity) use OAuth 2.0 for their integration APIs. Essential for any platform exposing a developer API to enterprise customers.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Standard for representing claims securely between parties as JSON objects. Used alongside OAuth 2.0 for bearer-token authentication in supply chain API integrations. Nearly all modern supply chain API implementations issue JWTs as access tokens.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication layer built on top of OAuth 2.0. Enterprise supply chain platforms require Single Sign-On (SSO) integration; OpenID Connect is the de-facto mechanism. All enterprise SaaS platforms in this space support OIDC for identity federation with customer identity providers.

**RFC 7807 — Problem Details for HTTP APIs**
- URL: https://www.rfc-editor.org/rfc/rfc7807
- Standard format for returning machine-readable error details from HTTP APIs. Adoption reduces integration friction; supply chain platform APIs should return structured error payloads using this format to simplify debugging by integration partners.

---

### Data Model & API Specifications

**GS1 EPCIS 2.0 / CBV 2.0 — Electronic Product Code Information Services**
- URL: https://www.gs1.org/standards/epcis
- Implementation guideline: https://ref.gs1.org/guidelines/epcis-cbv/
- Ratified by GS1 in June 2022. The dominant standard for capturing and sharing supply chain event data (what, when, where, why, how of product movement). EPCIS 2.0 adds REST/JSON API support alongside legacy XML/SOAP, sensor data capture, and reduced adoption barriers. Essential for any supply chain visibility or traceability features integrating with trading partners. The FDA's FSMA Rule 204 food traceability requirements reference EPCIS as the recommended data format.

**GS1 Standards — GTIN, GLN, SSCC, EDI**
- URL: https://www.gs1.org/standards
- Universal product (GTIN), location (GLN), and shipment (SSCC) identifiers underpinning supply chain data interoperability. Any platform ingesting supplier or logistics data will encounter GS1 identifiers in inbound feeds. Correct handling of GTINs and GLNs is a prerequisite for trading-partner integration features.

**ANSI X12 EDI (ASC X12)**
- URL: https://x12.org/products/transaction-sets
- The dominant B2B electronic data interchange standard in North America. Key supply chain transaction sets:
  - 850 — Purchase Order
  - 855 — Purchase Order Acknowledgement
  - 856 — Advance Ship Notice (ASN)
  - 810 — Invoice
  - 997 — Functional Acknowledgement
  - 940 — Warehouse Shipping Order
  - 945 — Warehouse Shipping Advice
  These five to seven transaction sets represent ~90% of retail supply chain EDI volume. Any platform offering supplier data ingest must handle X12 or delegate to an EDI middleware partner.

**UN/EDIFACT**
- URL: https://unece.org/trade/uncefact/introducing-unedifact
- The international equivalent of X12 EDI, dominant in Europe and Asia. Key EDIFACT message types: ORDERS (purchase order), DESADV (despatch advice / ASN), INVOIC (invoice), APERAK (application acknowledgement). European supply chain deployments typically use EDIFACT rather than X12.

**UN/CEFACT Supply Chain Reference Data Model (SCRDM)**
- URL: https://www.unescap.org/sites/default/files/Session%202_SCRDM_UNCEFACT.pdf
- Package: https://unece.org/trade/documents/2024/07/informal-documents/uncefact-package-standards-data-exchange-along-supply
- The UN/CEFACT 2024 standards package for seamless electronic multimodal data and document exchange. Exchange-syntax independent; designed for implementation in XML, JSON, RESTful APIs, and emerging technologies including blockchain. Referenced by the EU eFTI Regulation (2020/1056) for electronic freight transport information. Important reference for any multi-modal logistics analytics module.

**OpenAPI Specification 3.1 / 3.2**
- URL: https://spec.openapis.org/oas/v3.2.0.html
- The dominant standard for describing HTTP/REST APIs. Version 3.2.0 (released September 2025) adds structured tag nesting, streaming media type support (SSE, JSON Lines), native QUERY method support, and OAuth 2.0 Device Authorization Flow. All public-facing supply chain platform APIs should be described in OpenAPI 3.1+ for developer tooling compatibility. Kinaxis, Blue Yonder, and AWS Supply Chain all publish or reference OpenAPI specifications.

**AsyncAPI Specification 3.x**
- URL: https://www.asyncapi.com/docs/reference/specification/v3.1.0
- The standard for describing event-driven / asynchronous APIs (Kafka topics, WebSockets, AMQP, MQTT). Increasingly adopted for supply chain real-time messaging: European logistics carriers are migrating from ad-hoc webhooks to AsyncAPI-defined event streams. A supply chain analytics platform implementing real-time inventory updates or disruption alerts via Kafka/message queues should publish an AsyncAPI document for integration partners.

---

### Process Reference Models

**SCOR Digital Standard (ASCM/APICS)**
- URL: https://scor.ascm.org/
- Overview: https://www.ascm.org/corporate-solutions/standards-tools/scor-ds/
- The Supply Chain Operations Reference (SCOR) Digital Standard is the cross-industry process reference model for supply chain management, maintained by ASCM (formerly APICS). Defines the six process domains: Plan, Source, Make, Deliver, Return, Enable. Provides standardised performance metrics (perfect order fulfilment, supply chain cycle time, upside supply chain flexibility, etc.). Enterprise customers universally use SCOR metrics for benchmarking; a supply chain analytics platform should align its KPI definitions to SCOR vocabulary to accelerate adoption.

---

### Regulatory Frameworks

**EU CSDDD — Corporate Sustainability Due Diligence Directive**
- URL: https://www.corporate-sustainability-due-diligence-directive.com/
- Enacted July 2024; member state transposition deadline July 2027; compliance obligations begin 2028 (companies ≥3,000 employees, >€900M turnover) and 2029 (≥1,000 employees, >€450M). Requires ongoing identification, prevention, and mitigation of adverse human rights and environmental impacts across supply chains. Creates hard regulatory demand for supplier risk scoring, ESG due diligence modules, and annual public reporting features in any platform targeting EU enterprise customers.

**German LkSG — Lieferkettensorgfaltspflichtengesetz (Supply Chain Due Diligence Act)**
- URL: https://www.bmas.de/EN/Europe-and-the-World/International/Supply-Chain-Act/supply-chain-act.html
- In force since January 2023 for companies ≥3,000 employees; extended to ≥1,000 employees from January 2024. Precursor to CSDDD; requires annual risk analysis and corrective-action reporting across direct and indirect supplier tiers. German-headquartered enterprises and companies with significant German operations are already subject to LkSG obligations, creating immediate market demand.

**EU GDPR — General Data Protection Regulation**
- URL: https://gdpr.eu/
- Governs the processing of personal data in the EU. Supply chain platforms processing supplier contact data, employee records, or customer order data must comply with GDPR data minimisation, purpose limitation, and data subject rights requirements. API designs must support data erasure and portability endpoints for GDPR compliance.

---

### Security Standards

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- The authoritative reference for API security vulnerabilities. Supply chain API endpoints handling supplier financial data, inventory levels, and procurement data are attractive attack targets. Platform APIs must address the OWASP Top 10 risks: broken object-level authorisation, broken authentication, excessive data exposure, lack of rate limiting, and others.

**ISO 27001:2022 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- The internationally recognised ISMS standard. Enterprise supply chain customers require ISO 27001 certification from SaaS vendors as a procurement prerequisite. Flowlity explicitly holds ISO 27001 certification as a competitive differentiator in mid-market sales.

---

## Similar Products — Developer Documentation & APIs

### AWS Supply Chain

- **Description:** AWS-native supply chain visibility and demand planning service with pay-as-you-go pricing. Integrates natively with AWS data services and announced agentic AI capabilities in 2026.
- **API Documentation:** https://docs.aws.amazon.com/aws-supply-chain/latest/APIReference/Welcome.html
- **SDKs/Libraries:**
  - Python (boto3): https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/supplychain.html
  - Go SDK: https://docs.aws.amazon.com/sdk-for-go/api/service/supplychain/
  - AWS SDKs available for Java, .NET, JavaScript, and others via https://aws.amazon.com/tools/
- **Developer Guide:** https://docs.aws.amazon.com/aws-supply-chain/
- **Standards:** REST/JSON; AWS Signature Version 4 authentication; IAM-based access control
- **Authentication:** AWS IAM (Identity and Access Management); all operations require AWS-signed requests; no unauthenticated access

---

### SAP Integrated Business Planning (IBP)

- **Description:** SAP's integrated supply chain planning platform tightly coupled to SAP ERP/S4HANA. Dominant in SAP-committed enterprises for demand sensing, supply planning, and S&OP.
- **API Documentation:** https://api.sap.com/package/IBPAPIService/odata (SAP API Business Hub)
- **OData Services:**
  - `/IBP/PLANNING_DATA_API_SRV` — Read/write key figure (KPI) data
  - `/IBP/MASTER_DATA_API_SRV` — Read/write master data (products, locations, customers)
  - `/IBP/API_PRODUCTION` — Production OData API (see SAP KBA 3583208)
- **Developer Guide:** https://help.sap.com/docs/SAP_INTEGRATED_BUSINESS_PLANNING
- **Standards:** OData v2 REST; JSON and XML serialisation; Communication Arrangement SAP_COM_0720 required
- **Authentication:** OAuth 2.0 (CSRF token required for write operations); Basic Auth supported for legacy integrations

---

### Blue Yonder

- **Description:** Enterprise supply chain platform (Panasonic subsidiary) covering demand planning, inventory optimisation, transportation, and warehouse management. Market leader with AI Knowledge Graph and agentic AI capabilities.
- **API Documentation:** https://success.blueyonder.com/s/article/Open-Access-Web-API-Information
- **Developer Portal:** https://info.blueyonder.com/blue-yonder-platform/what-is-blue-yonder-connect-api-expansion-pack
- **SDKs/Libraries:** MuleSoft-based pre-built connectors via Blue Yonder Connect API & Expansion Pack; no public open-source SDK
- **Developer Guide:** Partner certification programme required; integration testing environment available to certified partners
- **Standards:** REST/JSON; OData; SOAP/XML for legacy systems; EDI (X12, EDIFACT) via Blue Yonder Connect
- **Authentication:** OAuth 2.0; enterprise SSO (SAML 2.0 / OIDC) for portal access

---

### Kinaxis Maestro (formerly RapidResponse)

- **Description:** Concurrent supply chain planning platform; industry benchmark for real-time scenario modelling. Agentic AI layer (Maestro) provides prescriptive planning recommendations.
- **API Documentation:** https://knowledge.kinaxis.com (requires customer/partner credentials)
- **MCP Server (AWS Marketplace):** https://aws.amazon.com/marketplace/pp/prodview-4osobep6cx3yc — Kinaxis MCP Server enabling AI agent integration via OAuth-secured Model Context Protocol
- **Developer Studio:** https://www.kinaxis.com/en/developer-studio — Low-code tool for custom apps, data model extensions, integrations, and visualisations
- **SDKs/Libraries:** Integration platform connectors for SAP, Oracle, Microsoft; community integration flows on GitHub: https://github.com/SimioLLC/KinaxisRapidResponse
- **Standards:** REST/JSON API; proprietary workbook data model; EDI X12/EDIFACT for trading partner data
- **Authentication:** OAuth 2.0 (MCP Server); proprietary session tokens for RapidResponse API

---

### RELEX Solutions

- **Description:** Supply chain and retail planning platform; strongest in demand forecasting for perishables, promotions, and high-velocity retail SKUs. Gartner Magic Quadrant Leader (2025).
- **API Documentation:** https://www.relexsolutions.com/api/monitoring-api-example-customer.html
- **Monitoring API GitHub:** https://github.com/relex/monitoring-api-demo
- **Developer Guide:** Integration documentation provided to customers; broader product APIs are not publicly documented
- **Standards:** REST/JSON (Monitoring API); file-based integration via SFTP; OpenAPI 3.x for Monitoring API
- **Authentication:** OAuth 2.0 (Monitoring API); API key for SFTP-based integrations

---

### o9 Solutions

- **Description:** AI-native integrated business planning platform using a graph-based data model. Supports demand, supply, and financial planning with multi-model ML ensemble forecasting.
- **API Documentation:** https://documents.o9solutions.com/
- **Developer Guide:** https://o9solutions.com/videos/integration-with-the-o9-platform-a-step-by-step-guide/
- **Standards:** REST API; SOAP/XML for legacy ERP; SFTP batch integration; supports Snowflake, Google BigQuery, SAP, Oracle, and Microsoft connectors
- **Authentication:** OAuth 2.0; SSO (SAML 2.0) for enterprise identity federation

---

### frePPLe

- **Description:** The only production-ready open-source supply chain planning platform. Community Edition (MIT licence from v8.0) covers statistical demand forecasting, MRP/DRP, and production scheduling. Enterprise Edition adds REST API, Gantt chart, and advanced demand planning.
- **API Documentation:** https://github.com/frePPLe/frepple/blob/master/doc/integration-guide/rest-api/api-from-the-command-line.rst
- **Source Code:** https://github.com/frePPLe/frepple
- **Developer Guide:** https://frepple.com (documentation, screencasts, and build instructions)
- **Standards:** REST/JSON API (Enterprise Edition); CSV import (Community Edition); PostgreSQL backend
- **Authentication:** Session-based authentication; Basic Auth for REST API; Enterprise Edition supports SSO

---

### Google OR-Tools

- **Description:** Google's open-source operations research library for vehicle routing (VRP), scheduling, bin-packing, and linear/integer programming. Used as a solver engine in custom supply chain applications.
- **API Documentation:** https://developers.google.com/optimization
- **Python Reference:** https://or-tools.github.io/docs/python/index.html
- **Source Code:** https://github.com/google/or-tools
- **SDKs/Libraries:** Python, Java, C#, C++ bindings; pip-installable (`pip install ortools`)
- **Developer Guide:** https://developers.google.com/optimization/introduction/python
- **Standards:** Apache 2.0 licence; no network API — library integrated directly into application code; outputs via solver result objects
- **Authentication:** Not applicable (local library)

---

### Flowlity

- **Description:** AI-driven supply chain planning SaaS targeting mid-market manufacturers and distributors. Probabilistic inventory buffers, automated safety stock optimisation, and fast ERP integration. Gartner Cool Vendor recognition.
- **API Documentation:** https://www.flowlity.com/tech/integration-security (integration overview; full API docs provided to customers)
- **Standards:** REST API (HTTPS); SFTP file transfer; pre-built connectors for SAP Business One, Microsoft Business Central, SAP S/4HANA, Oracle; ISO 27001:2022 certified
- **Authentication:** OAuth 2.0; customer-specific API credentials; enterprise SSO support

---

## Notes

**Emerging standard to watch — MCP (Model Context Protocol):**
Anthropic's Model Context Protocol (https://modelcontextprotocol.io/) was introduced in 2024 as an open standard for connecting AI agents to external tools and data sources. Kinaxis has already published an MCP Server on AWS Marketplace, enabling AI agents to interact with RapidResponse planning data via OAuth-secured MCP. As agentic AI capabilities become table-stakes for supply chain platforms, MCP compliance is likely to become an integration expectation for enterprise buyers by 2027.

**EDI modernisation gap:**
The majority of supplier data exchange still flows via EDI X12/EDIFACT, which requires specialist EDI middleware (TrueCommerce, SPS Commerce, Cleo) to parse and route. An AI-native supply chain platform should either partner with an EDI provider or build an EDI ingestion layer to avoid becoming dependent on the customer's existing EDI infrastructure.

**GS1 EPCIS 2.0 adoption:**
EPCIS 2.0's new REST/JSON interface significantly lowers the adoption barrier compared to legacy SOAP/XML EPCIS 1.2. FDA FSMA Rule 204 food traceability compliance (effective January 2026) is accelerating EPCIS 2.0 adoption in food and beverage supply chains — representing a near-term integration opportunity for any platform targeting that vertical.
