# Adobe Experience Platform & CX Enterprise — Consultant's Knowledge Snapshot

> **Snapshot date:** November 24, 2026
> **Audience:** Engineers and consultants who implement, architect, and advise on Adobe's customer-experience stack (AEP, Data Collection, Web/Mobile SDK, RT-CDP, AJO, CJA, Adobe Analytics, Target, AEM, and the new agentic layer).
> **How to use this:** Sections 1–2 are what changed recently. Sections 3–12 cover each product: how it works, what clients ask for, and where implementations go wrong. Section 13 lists deprecations. Section 14 is the source list.
>
> Facts marked **(2026)** come from Adobe release notes and press coverage checked on the snapshot date. Everything else is long-standing platform knowledge. Adobe ships monthly, so check the release-notes links in §14 before you quote a date to a client.

---

## 1. The big picture: what changed in 2025–2026

| Change | What it means for consulting |
|---|---|
| **Experience Cloud was renamed "Adobe CX Enterprise"** (Adobe Summit, April 20, 2026) **(2026)** | Client decks, SOWs, and RFP language now use "CX Enterprise." Product names (AEP, AJO, CJA, Target, AEM) stay the same. The core-services UI was refreshed with dark mode and accessibility improvements. |
| **CX Enterprise Coworker** (GA around June 10–11, 2026) **(2026)** | A persistent, goal-driven agentic layer that orchestrates Adobe and third-party agents. It **replaced AI Assistant and Data Insights Agent** in CJA and AEP docs. It runs in the product UIs and inside Microsoft 365 Copilot, ChatGPT, and Claude. It supports skills, plugins, and integrations, and is billed by credits. |
| **AEP Agent Orchestrator** (launched March 2025, agents GA in 2025) | The reasoning engine underneath the agents: Audience Agent, Journey Agent, Data Insights, Operational Insights, and others. Coworker sits on top of it. |
| **"Adobe CX Analytics" portfolio** (Summit 2026) **(2026)** | An umbrella over Adobe Analytics, CJA, CJA B2B Edition, Marketing Campaign Analytics, and Content Analytics. |
| **Adobe Brand Visibility** (formerly LLM Optimizer; Adobe closed its Semrush acquisition on April 28, 2026) **(2026)** | Generative engine optimization (GEO/AEO): how ChatGPT, Gemini, Perplexity, and similar tools cite a brand. It integrates with CJA to tie AI-driven discovery to on-site outcomes. This is a new advisory service line. |
| **Adobe Brand Concierge** **(2026)** | A conversational, agentic shopping and support experience on the brand's own site. It is wired into Web SDK (`conversation` config) and integrates with RT-CDP. |
| **Growth Credits** (AEP, RT-CDP, AJO) **(2026)** | A flexible way to buy capacity above contracted license limits. License-usage monitoring and overage management are now a recurring advisory need. |
| **Analytics 1.4 API and WSSE auth retired** (August 2026) **(2026)** | Any client integration still on 1.4 or WSSE is broken. Move it to the Analytics 2.0 API with OAuth server-to-server credentials in Adobe Developer Console. |
| **Coworker can migrate Adobe Analytics projects and components to CJA** (October 2026) **(2026)** | AA→CJA migration tooling is getting more automated. The architecture work is still the consultant's: schema, identity, and data views. |

---

## 2. Architecture mental model

```
            ┌────────────────────── CX Enterprise Coworker / Agent Orchestrator ──────────────────────┐
            │   (agents: Audience, Journey, Data Insights, Operational Insights, Product Support…)    │
            └──────────────────────────────────────────────────────────────────────────────────────────┘
 COLLECT                    PLATFORM (AEP)                                   ACTIVATE / ANALYZE / EXPERIENCE
 ────────                   ──────────────                                   ───────────────────────────────
 Web SDK (alloy.js) ──┐                                                       RT-CDP Destinations (ads, cloud, CRM)
 Mobile SDK ──────────┼─► Edge Network ─► Datastream ─┬─► AEP data lake ─┐    Journey Optimizer (journeys, campaigns,
 Server API ──────────┘   (edge segmentation,        │   (XDM datasets)   │      decisioning, channels)
 Sources (batch/stream)    personalization)          │                    ├─► Customer Journey Analytics
 Tags (Launch)                                        ├─► Adobe Analytics │    Adobe Target (via Edge)
 Event Forwarding (server-side) ◄────────────────────┤   (report suite)  │    AEM (content, Edge Delivery)
                                                      ├─► Adobe Target    │    Brand Concierge
                                                      └─► 3rd parties via │
                                                          Event Forwarding│
                                         Identity Service ─► Real-Time Customer Profile ─► Segmentation (batch/stream/edge)
                                         Query Service · Data Distiller · Data Governance (DULE labels, policies) · Privacy Service
```

The rules that drive almost every architecture decision:

1. **XDM is the contract.** Every dataset follows a schema built from a class (Individual Profile, ExperienceEvent, or a custom/relational class) plus field groups.
2. **Identity is the hardest part.** Choose namespaces (ECID, CRM ID, hashed email, and others), primary identities, and identity graph linking rules carefully. Most bad profiles trace back to a bad identity design.
3. **Datastreams route data.** One Edge call can fan out to AEP, Analytics, Target, AJO, Audience Manager, and Event Forwarding.
4. **Sandboxes** separate dev, stage, and prod. Promote configuration with sandbox tooling packages.
5. **Guardrails** cover profile fragment size, identity graph size, segments, destinations, and data views. Check them early in design, not at go-live.

---

## 3. Data Collection

### 3.1 Web SDK (alloy.js)
- **Latest version (2026): 2.37.1, released November 4, 2026.**
  - 2.37.0 added `advertising.enabled` to switch off Adobe Advertising at runtime.
  - 2.36.x added Brand Concierge surfaces and `keepalive` on `documentUnloading` events.
  - 2.35.x forwards the Marketo Munchkin cookie and checks the ECID on push notifications.
  - 2.34.0 added Brand Concierge `conversation.region`.
- **Core commands:** `configure`, `sendEvent`, `setConsent`, `getIdentity`, `appendIdentityToUrl`, `applyResponse` (hybrid/server-side), `getLibraryInfo`, `sendPushSubscription`, plus `conversation` for Brand Concierge.
- **Delivery:** use the Web SDK tag extension (most clients), a self-hosted CDN build, or npm (`@adobe/alloy`). Custom builds can drop unused components to shrink the bundle.
- **Key concepts:**
  - The `xdm` payload is validated against the schema. The `data` object is free-form and must be mapped in the datastream.
  - Personalization uses decision scopes and surfaces (`web://…`), with `renderDecisions` and `personalization.sendDisplayEvent`.
  - Identity uses the `identityMap` and ECID. `kndctr_*` cookies are first-party. A CNAME / first-party domain is still the best practice.
  - Consent uses the Adobe 2.0 standard or IAB TCF 2.x. Set `defaultConsent: "pending"` when a CMP is present.
  - Hide flicker with the prehiding snippet for Target/AJO web.
  - The `__alloyMonitors` monitoring hooks expose debugging and lifecycle events.
- **Migrating from AppMeasurement / at.js:** run the two side by side, map variables to XDM or the `data` object, use datastream mapping for Analytics (`data.__adobe.analytics.*`), and plan for ECID continuity. ID migration with `idMigrationEnabled` is on by default.

### 3.2 Mobile SDK
- Built from extensions: Core, Edge Network, Identity for Edge Network, Consent, Assurance, Lifecycle, Messaging (AJO push and in-app), Optimize (Target/AJO decisioning), and Edge Media.
- **(2026):** Android Edge 3.2.0 (October 21, 2026) sends device attributes (push tokens, timezone) to the new `mobile/device-attributes` endpoint, independent of collect consent. It requires AEPCore 3.11.0 or later.
- Use **Assurance (Project Griffon)** to validate events in real time on both mobile and web.

### 3.3 Tags (formerly Launch)
- Properties hold extensions, data elements, and rules (events, conditions, actions), published through Libraries and dev/stage/prod environments.
- Best practices:
  - Keep custom code to a minimum.
  - Use the Adobe Client Data Layer (ACDL) or a well-governed custom data layer.
  - Name rules and data elements consistently, and keep the rule count low.
  - Use the Reactor API for automation and audits.

### 3.4 Event Forwarding (server-side tags)
- A paid feature included in RT-CDP Connections, Prime, and Ultimate.
- Its properties are separate from Tags. Rules **have no event component**: they process every event on the datastream and use conditions to filter.
- There is a **30-second timeout** per event across all rules.
- Typical uses are Meta CAPI, Google, TikTok, webhooks, and enrichment calls (Secrets support tokens and OAuth). It reduces page weight and protects first-party data.

### 3.5 Datastreams
- Configured per environment. They choose the services (AEP datasets, Analytics report suites, Target property/environment, AJO, Event Forwarding, Audience Manager) and handle data prep mapping and configuration overrides (per-request report suite, dataset, or property). They also set geo-lookup and device-lookup settings and the first-party ID cookie.
- **(2026)** Datastream updates shipped in the March 2026 AEP release.

---

## 4. Adobe Experience Platform (AEP) core services

| Service | What it does | Consultant focus |
|---|---|---|
| **XDM / Schemas** | Classes, field groups, data types, and relational schemas. **(2026)** Relational schemas can now link to primary identity fields in standard schemas without segmentation enablement (Oct 2026). | Design schemas around use cases, avoid bloated field groups, and plan for profile enablement. |
| **Sources** | 100+ connectors: cloud storage, CRMs, databases, Snowflake, BigQuery, Kafka, Event Hubs, and the Analytics source connector. **(2026)** The Delta Sharing beta ended August 1, 2026 (GA planned for 2027). Azure Event Hubs VNet allowlisting is deprecated (Oct 2026). | Batch vs. streaming, data prep mapping, calculated fields, incremental loads. |
| **Identity Service** | Namespaces, identity graphs, graph linking rules (unique namespaces, priority). | Prevent graph collapse (shared devices, call-center IDs). Choose a primary identity per dataset. |
| **Real-Time Customer Profile** | Merges profile fragments using merge policies. **(2026)** Unstructured data support is in beta (Oct 2026). | Merge policies, profile store size and TTL, entitlement sizing. |
| **Segmentation Service / Audience Portal** | Batch, streaming, and edge evaluation. Audience Composition, external audiences, federated audiences. **(2026)** Built-in audience validation (Jan 2026), external audience expiration refresh, evaluation for Audience Compositions (beta, Oct 2026). | Choose the right evaluation method, follow streaming segment rules, and size audiences. |
| **Destinations** | Advertising, social, cloud storage, CRM, personalization (custom/Target/edge). **(2026)** "When to activate" triggers (attribute, segment, or identity changes) went GA in Oct 2026. Also: Azure Private Link, Snowflake Private Link, export of arrays as enrichment attributes, Google Ad Manager 360 GA, Google Customer Match + DV360 GA, 6sense Account (beta), The Trade Desk third-party audiences, UID2 conversion namespaces. | Destination SDK for custom destinations, activation cadence, governance policies on activation. |
| **Query Service / Data Distiller** | SQL over the data lake, derived datasets, scheduled queries, and the BI connection (PostgreSQL protocol). | Build derived attributes, decile models, and data-quality checks. |
| **Data Governance** | DULE labels, marketing-action policies, consent policies, attribute-based access control. | Label datasets early. The policy engine enforces at activation. |
| **Privacy Service** | GDPR/CCPA access and delete requests. **(2026)** Adds support for India's DPDP Act and sandbox-specific data lake privacy requests. | Automate privacy-request workflows through the API. |
| **Data lifecycle** | Dataset and experience-event TTL, record deletes. **(2026)** e-TTL scanning finds datasets with no expiration set (June 2026). | Cost and license control: set TTLs on high-volume event data. |
| **Run & Operate / Process Manager** **(2026)** | Inspect batch activation job schedules and troubleshoot. Process Manager is a new workflow feature (beta, Oct 2026). | Operational health dashboards for managed-service clients. |
| **Sandboxes & tooling** | Package and promote configuration. | Plan CI/CD for AEP configuration. |
| **Licensing** | Profile richness, addressable audience, Growth Credits **(2026)**, license-usage dashboards. | Prevent overages and advise on TTL and pseudonymous-profile expiration. |

**Related apps on AEP:** Federated Audience Composition (queries the warehouse in place: Snowflake, BigQuery, Databricks, Redshift, and others), Real-Time CDP Collaboration (clean-room-style partner collaboration), and RT-CDP B2B Edition (accounts, opportunities, buying groups).

---

## 5. Real-Time CDP (RT-CDP)

- **Tiers:** Foundation, Select, Prime, and Ultimate, plus B2C, B2B, and B2P editions. Connections is the "lite" collection and forwarding tier.
- **Typical use cases:**
  - Paid-media suppression and lookalikes
  - Cart and browse abandonment
  - Next-best-offer on the web through edge segmentation and the custom personalization destination
  - Call-center profile lookup
  - First-party and identity-based ad targeting (UID2, RampID, hashed email)
- **(2026):** Brand Concierge integration, audience-level reporting for Facebook, TikTok, and other high-usage destinations, and Coworker audience skills (inventory distribution and health, lineage, duplicate detection, "why did my audience size change").
- **Common pitfalls:**
  - Pushing every attribute into the profile.
  - No TTL on events.
  - Ignoring streaming vs. batch latency.
  - Activating unlabeled data.
  - Weak identity design.

---

## 6. Adobe Journey Optimizer (AJO)

- **What it is:** native AEP orchestration covering real-time journeys (event-, audience-, and business-event-triggered), **campaigns** (scheduled or API-triggered, plus **orchestrated campaigns**, the multi-step, Campaign-style batch workflows), channels (email, SMS, push, in-app, web, code-based experience, content cards, WhatsApp, direct mail), and **Decisioning**.
- **Decisioning:** the older "Decision Management / Offer Decisioning" is being replaced by the new **Experience Decisioning** (decision items, collections, selection strategies, ranking, AI models). Migration APIs exist.
  - **(2026)** Personalized optimization models can use up to 50 profile attributes and 20 context attributes (LA, Nov 2026).
- **2026 highlights:**
  - **Jan:** native and Adobe Campaign v7/v8 message actions can be combined in one journey.
  - **Feb:** `executionMetadata` helper went GA.
  - **Apr:** The **Optimize activity replaced the Condition activity** (GA). Conditional paths, experiments, and targeting now live in one activity, so update training material accordingly.
  - **Jul:** Circuit breaker on slow custom-action endpoints (more than 20% of calls over 5 seconds within 120 seconds, given at least 200 calls, triggers a cap of up to 5 minutes). Also Guided Adoption migration workspace (LA).
  - **Oct:** On-demand journey validation and GA of custom outbound channels.
  - **Nov:** AI recommendation cards for journey alerts (LA), Coworker skills for orchestrated campaigns (read-only), transactional email migration into API-triggered campaigns, Coworker decision-item creation, dynamic token substitution in AEM Content Fragments (`{{profile.person.name.firstName}}`).
  - **Journey Agent / Coworker:** create, simulate, analyze, and compare journey versions, and build a journey from a flowchart image.
- **Engineering concerns:**
  - Custom actions need rate limiting, capping, and throttling.
  - Know the journey guardrails: re-entrance, wait nodes, profile throughput.
  - Configure channel surfaces: IP warming, subdomain delegation, DMARC/BIMI.
  - Plan consent and frequency rules, sandboxes, and the test-profile strategy.
- **AJO B2B Edition** (formerly Marketo-adjacent; buying groups, account journeys) **(2026):** static and dynamic people lists, split-path A/B nodes (beta), multi-trigger "Listen for an event" nodes.
- **AJO Loyalty** was announced at Summit 2026 for gamified loyalty programs.

---

## 7. Customer Journey Analytics (CJA)

- **Core model:**
  - **Connections** combine AEP datasets (event, profile, lookup, summary) on a person ID.
  - **Data views** define components, settings, derived fields, persistence, and attribution at query time, so changes apply retroactively.
  - **Analysis Workspace** is where reporting happens.
- **Stitching:** field-based and graph-based stitching combine authenticated and anonymous data. Replay windows matter. See the **Bad IDs (BAVIDs)** technote (Mar 2026).
- **Key features:**
  - Derived fields, with a guidelines and pitfalls article (Apr 2026).
  - Journey canvas, Guided analysis, Experimentation panel, Content Analytics, Report Builder, BI extension (Power BI, Tableau, Looker, Jupyter, RStudio).
  - Full-table exports to the cloud.
  - Audience publishing to RT-CDP (refresh limits of 75–150 audiences, depending on entitlement).
  - Data Mirror (mirrors native warehouse tables).
  - **CJA B2B Edition** (person and account containers).
- **2026 highlights:**
  - **CJA MCP server** with **MCP Read-only** vs. **MCP Full Access** permissions (Oct 6, 2026). You can now point AI clients at CJA safely.
  - **Conversation Insights:** collect LLM and agent prompts, responses, and metadata through Web SDK, then analyze intent, tone, and sentiment and tie them to outcomes. Rollout started Oct 29, 2026.
  - Brand Visibility integration (Oct 8), Journey canvas percentage-point comparisons (Oct 19), concise legend labels, auto-generated component and project descriptions (GA Nov 18).
  - In-product Coworker panel in Workspace, which can summarize, run root-cause analysis, and build or edit projects.
  - New CJA API guides: advanced date-trended reports, breakdowns, recurring reports, data view ID best practices.
  - Sub-event analysis and custom containers, inline classifications (Jul).
  - Paid media channel in Content Analytics (Google Ads and Meta Ads source connectors).
  - Guardrail changes: data views per org rose from 2,000 to 5,000; calculated metrics per org rose to 50,000.
  - Person-scoped metric deduplication now applies only within the reporting window (Nov 2026).
  - New **GA4 → CJA transition guide** (Jun 2026).
- **Postponed:** total population reporting (entity reporting on profile and lookup datasets) and Streaming Media schedule data.

### AA → CJA migration playbook (high level)
1. Run an **org-readiness assessment** (Adobe has a guide), build a use-case inventory, and audit AA components and usage.
2. **Pick a data path:** Web SDK into AEP (best long term) or the Analytics source connector (fastest, but carries AA's structure along).
3. **Design the schema** to fit CJA, not to copy eVars and props. Adobe's Jan 2026 "Architect your schema" guide covers this.
4. Plan **identity and stitching**, then build **data views** (persistence and attribution replace eVar allocation and expiration).
5. **Rebuild components.** Coworker can now migrate AA projects and components (Oct 2026), but validate every one.
6. Run AA and CJA in parallel, reconcile the numbers, train users, then retire AA.

---

## 8. Adobe Analytics (AA)

- Still sold and developed. There is no announced end-of-life, and it is part of the CX Analytics portfolio.
- **Core concepts:**
  - Report suites (and virtual report suites); eVars (allocation and expiration), props, events, list vars, merchandising eVars.
  - Classifications, processing rules, VISTA, marketing channels.
  - Data Feeds, Data Warehouse, Data Sources, Activity Map, A4T.
  - Implementation through AppMeasurement, the Analytics tag extension, or Web SDK (via the datastream).
- **2026:**
  - Journey canvas came to AA (GA June 5) and gained comparisons (Aug 18).
  - Fallout visualization improvements (Mar).
  - Coworker chat for root-cause analysis (Oct).
  - Data Warehouse classification improvements (Oct).
  - TLS 1.2 CBC cipher suites removed from data collection servers (May 27, 2026). This affects legacy native apps.
  - **1.4 API and WSSE retired (August 2026).**
- **API:** use Analytics 2.0 API with OAuth server-to-server credentials. JWT credentials were deprecated earlier, so move clients to OAuth S2S.

---

## 9. Adobe Target

- **Activity types:** A/B, Auto-Allocate, Auto-Target, Automated Personalization, Experience Targeting, Multivariate, Recommendations.
- **Authoring:** VEC, Form-Based Composer, EEC.
- **Delivery options:**
  - **at.js 2.x** (legacy client-side)
  - **Web SDK** (the strategic path)
  - Server-side Node.js/Java/.NET/Python SDKs, including **on-device decisioning**
  - Delivery API, and Adobe Target via the Edge (datastream)
- **Integrations:** A4T (Analytics for Target), Target + CJA reporting, AEP audiences and profile attributes through edge segmentation and the custom personalization destination, AEM Experience Fragments as offers.
- **2026 highlights:**
  - Enhanced Experience Composer status indicator (26.4.1, Apr).
  - AI Insights for manual A/B (beta, Sep 30).
  - **Sample Size Calculator** (beta, 26.9.8).
  - Analytics for VEC experiences (26.10.1).
  - **Safari support** for the Target UI, though the VEC and EEC still require Chrome (26.10.2).
  - **Coworker entry points**: "Review my live tests" and "Ask Coworker" on each activity (26.11.1, Nov 12).
- **at.js → Web SDK migration gotchas:**
  - Mbox names become decision scopes; `__view__` is the global scope.
  - Profile scripts and mbox parameters move to `data.__adobe.target`.
  - Prehiding works differently.
  - A4T has to be configured through the datastream.
  - Response tokens and third-party ID (`mbox3rdPartyId`) are mapped through the `identityMap` and the datastream's Target third-party ID namespace.

---

## 10. Adobe Experience Manager (AEM): what a data/martech consultant needs

- **AEM as a Cloud Service (AEMaaCS):**
  - Monthly feature releases (2026.10.0 is current and 2026.11.0 was planned for Nov 19, 2026), Cloud Manager CI/CD pipelines, Dispatcher.
  - The local SDK supports Java 25 (Oct 2026).
  - OpenTelemetry APM is in LA.
  - Cloud Manager 2026.11.0 added web-tier pipelines for AMS.
- **Edge Delivery Services** (document-based or Universal Editor authoring; very fast sites) is the strategic direction for new sites.
- **AEM Assets / Content Hub / Dynamic Media**:
  - **(2026)** AI captions, AI search, and Smart Collections.
  - **GenStudio for Performance Marketing** sits next to it for generative content supply chain work.
- **AEM Forms; AEM Sites + Commerce (CIF).**
- **Agents (2026):** AEM Governance Agent and Coworker Dispatcher and workflow inspection. **LLM Apps** let brands build experiences inside LLM interfaces.
- **Touchpoints with the data stack:**
  - Web SDK and the data layer in page templates.
  - Experience Fragments and Content Fragments feeding Target and AJO.
  - Content Analytics in CJA.
  - Brand Visibility auto-deploy at the edge.

---

## 11. Other products in the portfolio you'll be asked about

- **Adobe Campaign v8** (Classic/Standard): batch, complex multichannel campaigns. It can now be combined with AJO in journeys.
- **Marketo Engage**: B2B marketing automation. **(2026)** Coworker "Plan Campaigns" skill, Callable Agents as smart campaign flow steps (coming soon), new UI.
- **Adobe Commerce** (Magento): catalog enrichment for AI shopping. **(2026)**
- **Workfront / GenStudio**: content supply chain.
- **Adobe Mix Modeler, Marketing Campaign Analytics**: measurement and MMM.
- **Audience Manager**: legacy DMP. Most clients are moving to RT-CDP.
- **Brand Visibility, Brand Concierge**: see §1. **(2026)**

---

## 12. Cross-cutting consulting knowledge

### 12.1 Identity & consent checklist
- Use first-party CNAME data collection.
- Keep ECID continuity across AppMeasurement, at.js, and Web SDK.
- Decide whether to use FPID (a server-generated first-party device ID) for resilience against ITP.
- Map the CMP into `setConsent` (Adobe 2.0 or TCF) and test both the default-pending and denied paths.
- Hash PII (SHA-256) before it is used for ad destinations.

### 12.2 Implementation validation toolkit
- **Adobe Experience Platform Debugger** (browser extension).
- **Assurance** for web and mobile.
- Edge trace in Assurance.
- Datastream and dataset monitoring, plus batch ingestion error diagnostics.
- Query Service data-quality SQL.
- Coworker data validation skill for CJA datasets **(2026)**.

### 12.3 APIs & automation
- **Auth:** Adobe Developer Console with OAuth Server-to-Server. JWT is deprecated.
- **APIs:**
  - AEP: Catalog, Schema Registry, Flow Service, Segmentation, Destinations, Query, Sandbox Tooling, Privacy.
  - Reactor API (Tags).
  - Analytics 2.0, CJA API, Target Admin/Delivery API, AJO APIs (including Decisioning migration).
- **MCP servers:** CJA MCP server (2026). Coworker is also available in external AI platforms.
- **Infrastructure as code:** sandbox packages, Reactor API scripts, community Terraform providers.

### 12.4 Common client pain points (and the service you can sell for each)
| Pain | Offer |
|---|---|
| "Our AEP data is a mess / profiles are wrong" | Identity and schema health assessment |
| "We bought RT-CDP but only activate one audience" | Use-case roadmap and activation accelerator |
| "AA is legacy, we need CJA" | AA→CJA migration program |
| "Personalization is slow or flickers" | at.js→Web SDK + edge decisioning migration |
| "We're over our license" | License-usage / Growth Credits optimization (TTLs, pseudonymous profile expiry) |
| "What do we do with Coworker / agents?" | Agentic AI enablement: skills, plugins, governance, MCP |
| "How do we show up in ChatGPT?" | Brand Visibility / GEO program + Conversation Insights measurement |
| "Our journeys break / custom actions time out" | AJO operational health review (capping, circuit breakers, alerts) |

---

## 13. Deprecations & EOL watchlist (as of Nov 2026)

| Item | Status |
|---|---|
| Analytics 1.4 API + WSSE auth | **Retired.** The release notes list August 12, 2026 and the discontinued-features page says "effective August 31, 2026." Either way it is gone. |
| JWT service-account credentials | Deprecated. Use OAuth S2S. |
| AI Assistant / Data Insights Agent (as named features) | Replaced by CX Enterprise Coworker. |
| AJO Condition activity | Removed from the UI and replaced by the Optimize activity (Apr 2026). |
| AJO Decision Management (Offer Decisioning) | Migrate to Experience Decisioning. |
| TLS 1.2 CBC cipher suites on Adobe collection servers | Removed May 27, 2026. |
| Azure Event Hubs VNet allowlisting (AEP source) | Deprecated (Oct 2026). |
| at.js / AppMeasurement | Not EOL, but Web SDK is the strategic path. |
| Audience Manager | Legacy. RT-CDP is the successor. |

---

## 14. Primary sources (bookmark these)

- CX Enterprise central release notes: https://experienceleague.adobe.com/en/docs/release-notes/experience-cloud/current
- AEP release notes (latest): https://experienceleague.adobe.com/en/docs/experience-platform/release-notes/latest
- AEP pre-release notes: https://experienceleague.adobe.com/en/docs/experience-platform/release-notes/pre-release-notes
- Web SDK release notes: https://experienceleague.adobe.com/en/docs/experience-platform/collection/js/release-notes
- Mobile SDK release notes: https://developer.adobe.com/client-sdks/home/release-notes/
- Event forwarding overview: https://experienceleague.adobe.com/en/docs/experience-platform/tags/event-forwarding/overview
- CJA release notes: https://experienceleague.adobe.com/en/docs/analytics-platform/using/releases/latest
- CJA doc changes: https://experienceleague.adobe.com/en/docs/analytics-platform/using/releases/doc-changes
- Adobe Analytics release notes: https://experienceleague.adobe.com/en/docs/analytics/release-notes/latest
- Analytics 1.4 EOL: https://experienceleague.adobe.com/en/docs/discontinued/using/analytics-1-4-apis
- Target release notes: https://experienceleague.adobe.com/en/docs/target/using/release-notes/release-notes
- AJO release notes: https://experienceleague.adobe.com/en/docs/journey-optimizer/using/whats-new/release-notes
- AJO B2B release notes: https://experienceleague.adobe.com/en/docs/journey-optimizer-b2b/user/release-notes
- AJO Decisioning migration: https://experienceleague.adobe.com/en/docs/journey-optimizer/using/decisioning/experience-decisioning/migrate-to-decisioning/migrate-to-decisioning
- Agent Orchestrator: https://experienceleague.adobe.com/en/docs/cx-enterprise-ai/experience-cloud-ai/agents/agent-orchestrator
- Coworker docs: https://experienceleague.adobe.com/en/docs/coworker/content/home
- CJA MCP server permissions: https://developer.adobe.com/analytics-mcp/docs/guides/permissions
- Summit 2026 / CX Enterprise press release: https://news.adobe.com/news/2026/04/adobe-redefines-custome-experience
- MarTech coverage of the rebrand: https://martech.org/adobe-rebrands-experience-cloud-as-cx-enterprise-goes-all-in-on-ai-agents/
- Brand Visibility announcement: https://news.adobe.com/news/2026/04/adobe-introduces-brand-visibility-solution
