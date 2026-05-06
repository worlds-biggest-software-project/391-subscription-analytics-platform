# Standards & API Reference

> Project: Subscription Analytics Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### Accounting & Revenue Recognition Standards

**ASC 606 — Revenue from Contracts with Customers (FASB)**
- URL: https://fasb.org/standards/asc/606
- The US GAAP standard governing how subscription businesses recognise revenue. Requires a five-step model: identify contract, identify performance obligations, determine transaction price, allocate price, recognise when obligation is satisfied. Any analytics platform operating in the US market must produce reports aligned with ASC 606, recognising MRR on a periodic delivery basis rather than at contract signing.

**IFRS 15 — Revenue from Contracts with Customers (IASB)**
- URL: https://www.ifrs.org/issued-standards/list-of-standards/ifrs-15-revenue-from-contracts-with-customers/
- The international counterpart to ASC 606, issued jointly by FASB and IASB to harmonise revenue recognition globally. SaaS companies selling across borders must reconcile both standards. Differences in collectibility thresholds (75–80% for ASC 606 vs 50% for IFRS 15) and treatment of contract costs affect how deferred revenue and MRR are reported.

### Security & Compliance Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The baseline information-security management standard. Subscription analytics platforms store sensitive billing data and financial metrics; ISO 27001 certification signals enterprise-grade controls. Major vendors including ChartMogul, Baremetrics, and Zuora hold or target this certification.

**ISO/IEC 27701:2019 — Privacy Information Management (extension to ISO 27001)**
- URL: https://www.iso.org/standard/71670.html
- Extends ISO 27001 with a privacy information management framework, directly relevant to GDPR and CCPA compliance when storing customer PII alongside subscription records.

**SOC 2 Type II (AICPA Trust Services Criteria)**
- URL: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- The de-facto enterprise audit standard for SaaS products. Covers Security, Availability, Processing Integrity, Confidentiality, and Privacy trust service criteria. B2B SaaS buyers routinely require a current SOC 2 Type II report before granting data-pipeline access to a subscription analytics platform.

**PCI DSS v4.0 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/standards/pci-dss/
- Governs handling of cardholder data. Subscription analytics platforms that ingest raw payment-processor events (Stripe, Braintree) must either maintain PCI DSS compliance or explicitly tokenise and exclude cardholder data at ingestion, relying on compliant processors for the raw payment layer.

### Privacy & Data Regulations

**GDPR — General Data Protection Regulation (EU 2016/679)**
- URL: https://gdpr.eu/
- Requires lawful basis for processing subscription customer PII, Data Processing Agreements (DPAs) with sub-processors, right-to-erasure workflows, and breach notification within 72 hours. Analytics platforms must support customer data deletion that cascades through historical cohort records without corrupting aggregate metrics.

**CCPA / CPRA — California Consumer Privacy Act & California Privacy Rights Act**
- URL: https://oag.ca.gov/privacy/ccpa
- US state-level privacy regulation requiring opt-out rights for data sale, data subject access requests, and data minimisation. Eight additional US state privacy laws took effect in 2025 (Delaware, Iowa, New Hampshire, New Jersey, Tennessee, Minnesota, Maryland, Kentucky), creating a patchwork that subscription analytics vendors must track.

### API & Data Format Standards

**OpenAPI Specification v3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.1.html
- The standard for documenting REST APIs. OAS 3.1 aligns fully with JSON Schema Draft 2020-12 and natively supports webhook definitions alongside path definitions. All major subscription analytics and billing platforms (Stripe, Chargebee, Recurly, Zuora) publish OAS-compatible API references. An open-source analytics platform should ship an OpenAPI document for its REST surface.

**AsyncAPI Specification v3.x**
- URL: https://www.asyncapi.com/docs/concepts/asyncapi-document
- The emerging standard for documenting event-driven and webhook APIs — the async counterpart to OpenAPI. Directly applicable for defining subscription billing event streams (customer.created, subscription.upgraded, invoice.payment_failed, etc.) across transports such as webhooks, Kafka, AMQP, and WebSockets.

**CloudEvents v1.0 (CNCF)**
- URL: https://cloudevents.io/ / https://github.com/cloudevents/spec
- A CNCF specification for describing event data in a common, vendor-neutral format. Defines envelope attributes (source, type, time, datacontenttype) and supports JSON, Protobuf, and Avro serialisation. Adopting CloudEvents for billing event payloads enables interoperability with AWS EventBridge, Google Eventarc, Azure Event Grid, and open-source brokers.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/draft/2020-12
- Used to formally specify the data models for subscription objects (customer, plan, subscription, invoice, event). OpenAPI 3.1 and AsyncAPI 3.x both use this dialect natively, making it the lingua franca for schema validation across the platform's API surface.

### Authentication & Authorisation Standards

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational standard for delegated API authorisation. All major subscription analytics platforms use OAuth 2.0 for third-party integrations (connecting Stripe or Chargebee accounts). An open-source platform should support OAuth 2.0 for partner integrations and for its own public API.

**RFC 9700 — OAuth 2.0 Security Best Current Practice**
- URL: https://www.rfc-editor.org/rfc/rfc9700
- Deprecates the Implicit grant and Resource Owner Password Credentials grant; mandates Authorization Code + PKCE for public clients. Informs how the platform should implement its OAuth flows.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0, used for user authentication in SaaS platforms. Enables SSO and integration with enterprise identity providers (Okta, Auth0, Azure AD).

**RFC 7617 — HTTP Basic Authentication**
- URL: https://datatracker.ietf.org/doc/html/rfc7617
- Still widely used by subscription analytics APIs (ChartMogul, Baremetrics) for simple API key authentication. An open-source platform should support both API key (Basic Auth) and OAuth 2.0 depending on the integration context.

---

## Similar Products — Developer Documentation & APIs

### ChartMogul

- **Description:** Purpose-built subscription analytics platform offering MRR segmentation, cohort net-MRR retention, benchmarking, and a CRM layer. The de-facto reference implementation for the category.
- **API Documentation:** https://dev.chartmogul.com/docs/introduction/
- **SDKs/Libraries:**
  - Python: https://github.com/chartmogul/chartmogul-python (Python 3.10–3.14)
  - Ruby: https://github.com/chartmogul/chartmogul-ruby
  - Go: https://github.com/chartmogul/chartmogul-go (Go 1.21+)
  - Node.js: https://github.com/chartmogul/chartmogul-node
  - PHP: https://github.com/chartmogul/chartmogul-php
- **Developer Guide:** https://dev.chartmogul.com/docs/getting-started-with-the-import-api
- **Standards:** REST/JSON, HTTP Basic Auth (API key)
- **Authentication:** HTTP Basic Authentication with API key

### Baremetrics

- **Description:** Subscription analytics and dunning tool pulling metrics directly from payment processors. Known for one-click Stripe integration, transparent benchmarks, and trial-to-paid conversion funnels.
- **API Documentation:** https://developers.baremetrics.com/reference/introduction
- **SDKs/Libraries:**
  - Ruby gem: https://github.com/rewindio/baremetrics_api
  - PHP: https://github.com/oseintow/baremetrics-api
- **Developer Guide:** https://developers.baremetrics.com/reference/quick-start
- **Standards:** REST/JSON, versioned endpoints (`/v1/:source_id/`)
- **Authentication:** Bearer token (API key in Authorization header)

### ProfitWell Metrics (Paddle)

- **Description:** Free-tier subscription analytics platform acquired by Paddle. Covers MRR, churn, LTV, and ARPU with bank-level accuracy. Enrichment and sharing API allows piping metrics into other tools.
- **API Documentation:** https://developer.paddle.com/concepts/profitwell-metrics
- **SDKs/Libraries:** Paddle SDK covers ProfitWell Metrics for Paddle Billing customers — https://github.com/paddlehq
- **Developer Guide:** https://www.paddle.com/help/profitwell-metrics/setup/get-started/profit-well-api
- **Standards:** REST/JSON
- **Authentication:** API key

### Stripe Billing & Revenue Recognition

- **Description:** Stripe's billing stack powers subscription creation and lifecycle management; the Revenue Recognition module automates ASC 606/IFRS 15 accrual accounting and exposes subscription-level MRR and churn metrics.
- **API Documentation:** https://docs.stripe.com/api / https://docs.stripe.com/billing / https://docs.stripe.com/revenue-recognition/api
- **SDKs/Libraries:** Official SDKs in Python, Ruby, Node.js, Java, PHP, Go, .NET — https://docs.stripe.com/libraries
- **Developer Guide:** https://docs.stripe.com/billing/subscriptions/overview / https://docs.stripe.com/billing/subscriptions/analytics
- **Standards:** REST/JSON, OpenAPI-compatible, webhook events via HTTPS POST
- **Authentication:** API key (secret key in Authorization Bearer header); webhook signatures via HMAC-SHA256

### Chargebee

- **Description:** Subscription billing and revenue operations platform. Comprehensive API covering subscriptions, invoices, usage metering, and revenue recognition. Widely used as a billing source for subscription analytics layers.
- **API Documentation:** https://apidocs.chargebee.com/docs/api/subscriptions
- **SDKs/Libraries:** Server-side SDKs in Java, Python, Ruby, Node.js, PHP, Go, .NET — https://chargebee.com/docs/2.0/developer_resources.html
- **Developer Guide:** https://apidocs.chargebee.com/docs/api/getting-started
- **Standards:** REST/JSON, HTTP Basic Auth, webhooks via HTTPS POST
- **Authentication:** HTTP Basic Authentication with API key as username

### Recurly

- **Description:** Enterprise subscription management platform. REST API covers subscription lifecycle, plan management, coupon engines, and dunning. SDKs support native in-app payments on iOS and Android.
- **API Documentation:** https://recurly.com/developers/api/
- **SDKs/Libraries:** Ruby, Python, Node.js, .NET, Go, Java, iOS, Android — https://recurly.com/product/integration-methods/
- **Developer Guide:** https://recurly.com/developers/api/v2021-02-25/
- **Standards:** REST/JSON, versioned API (`v2021-02-25`), webhooks
- **Authentication:** API key (Basic Auth)

### Zuora

- **Description:** Enterprise-grade subscription billing and revenue management platform. Provides order delta metrics and subscription-level analytics via REST API. Used by large enterprises with complex multi-product bundles.
- **API Documentation:** https://developer.zuora.com/v1-api-reference/api
- **SDKs/Libraries:** Java, Python, C# SDKs — https://developer.zuora.com/
- **Developer Guide:** https://docs.zuora.com/en/zuora-platform/integration/apis/rest-api
- **Standards:** REST/JSON, OpenAPI-compatible
- **Authentication:** OAuth 2.0 (recommended) or API key

### Maxio (formerly SaaSOptics + Chargify)

- **Description:** Formed by the 2022 merger of Chargify and SaaSOptics. Combines subscription billing automation with SaaS financial operations including revenue recognition and MRR/ARR analytics. Targets mid-market B2B SaaS.
- **API Documentation:** https://developer.maxio.com/ (Chargify lineage) / https://www.maxio.com/
- **SDKs/Libraries:** Inherits Chargify API client libraries (Ruby, Python, PHP, Node.js)
- **Developer Guide:** Available via the Maxio developer portal
- **Standards:** REST/JSON, webhooks
- **Authentication:** API key

---

## Notes

**Emerging event streaming patterns:** The subscription analytics category is moving from polling-based data ingestion (periodic CSV imports or REST API pulls) toward real-time event streaming. Stripe, Chargebee, Recurly, and Zuora all expose webhook streams; CloudEvents and AsyncAPI provide the standards vocabulary for documenting and normalising these streams. An open-source platform that adopts CloudEvents as its internal event envelope can more easily build adapters for Kafka, EventBridge, and other message brokers without locking into any single transport.

**Gap in open standards for SaaS metrics definitions:** There is no ISO or IETF standard defining how MRR, ARR, churn rate, LTV, or Net Revenue Retention should be calculated. ChartMogul, Baremetrics, and the SaaS community have developed informal de-facto definitions (e.g., the distinction between logo churn and revenue churn, or between gross MRR retention and net MRR retention), but these are not formally standardised. An open-source platform that publishes clear, machine-readable metric definitions (via JSON Schema or OpenAPI schemas) could itself become a reference implementation for the industry.

**Billing data source standards are fragmented:** Each major billing platform (Stripe, Chargebee, Recurly, Zuora) uses its own event taxonomy for subscription lifecycle events. There is no cross-vendor standard equivalent to the financial industry's ISO 20022 for subscription billing events. An open-source analytics platform must therefore ship adapters per source, and could advocate for or contribute to a common billing event schema built on CloudEvents.
