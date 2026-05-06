# Retail Store Analytics — Feature & Functionality Survey

> Candidate #399 · Researched: 2026-05-06

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| RetailNext | In-store analytics platform | Commercial SaaS | https://retailnext.net/ |
| V-Count / BoostBI | People counting + analytics cloud | Commercial SaaS | https://v-count.com/ |
| Sensormatic ShopperTrak | Retail traffic insights | Commercial SaaS (Johnson Controls) | https://www.sensormatic.com/shopper-insights |
| Dor Technologies | Thermal-sensing people counter + SaaS | Commercial SaaS | https://www.getdor.com/ |
| Placer.ai | Location intelligence / foot traffic | Commercial SaaS | https://www.placer.ai/ |
| Aislelabs | WiFi marketing + zone analytics | Commercial SaaS | https://www.aislelabs.com/ |
| Trakwell.ai | AI-vision retail analytics | Commercial SaaS | https://trakwell.ai/ |
| Lightspeed Retail Analytics | POS-integrated analytics | Commercial SaaS (POS add-on) | https://www.lightspeedhq.com/pos/retail/analytics/ |

---

## Feature Analysis by Solution

### RetailNext

**Core features**
- Real-time foot traffic counting using Aurora IoT device (AI deep-learning, samples 10×/second)
- Shopper journey analytics: full in-store path tracking across fixtures and zones
- Heatmaps showing dwell time and engagement by area
- POS and promotional calendar integration for contextual analysis
- Wi-Fi and video-based supplementary data capture
- Staff effectiveness and occupancy analysis
- Labour cost alignment: ties traffic data to scheduling and conversion rates
- High-resolution recorded video access for on-demand audit
- Multi-store reporting and benchmarking

**Differentiating features**
- Patent-pending Aurora sensor with anonymous deep-learning detection
- Test-and-learn capability: measure impact of store changes with A/B-style traffic comparisons
- Integration with weather services to contextualise traffic patterns
- 800 million+ shoppers analysed monthly across dozens of retail chains

**UX patterns**
- Dashboard-first interface aimed at retail operations teams
- Integrates with staffing systems and promotional tools for contextual overlay
- No SQL required for standard reporting

**Integration points**
- POS systems (unspecified, but multi-vendor support implied)
- Staff scheduling / labour management systems
- Weather services (third-party data enrichment)
- Promotional calendar feeds

**Known gaps**
- Full API documentation not publicly available; requires direct client engagement
- Hardware sensor lock-in (Aurora device proprietary)
- Pricing opaque, typically enterprise contract

**Licence / IP notes**
- Commercial SaaS; patent-pending on Aurora sensor algorithm. No open-source components publicly disclosed.

---

### V-Count / BoostBI

**Core features**
- 99%+ accurate AI people counting via Nano AI on-device sensor (3D depth sensing, works in darkness)
- Gender and age recognition (demographic analytics)
- Queue management and real-time occupancy (Vcare module)
- Zone-level heatmaps and dwell time analytics
- Storefront analytics (passerby vs. entrant conversion)
- Staff exclusion from counting for accurate visitor metrics
- BoostBI: cloud SaaS analytics dashboard with cross-location benchmarking
- Retailer and mall traffic index (market comparison)

**Differentiating features**
- AI-on-chip architecture: all processing on-device; only aggregated non-identifiable data transmitted
- Only vendor combining people counting, gender/age recognition, and staff exclusion in a single sensor
- 600+ customers including 11 Fortune 500 companies
- Explicit GDPR and privacy-by-design architecture

**UX patterns**
- BoostBI cloud dashboard for managers without technical skills
- Open REST API for integration with external BI tools

**Integration points**
- REST API (open) for third-party BI integration
- Existing tech stacks; API-first design

**Known gaps**
- Demographic analytics (gender/age recognition) faces increasing regulatory scrutiny under GDPR and EU AI Act
- Hardware-dependent; requires sensor purchase
- No native POS integration highlighted in available documentation

**Licence / IP notes**
- Commercial SaaS + hardware. Privacy-by-design architecture is a selling point, not a licensed open-source component.

---

### Sensormatic ShopperTrak

**Core features**
- Overhead people counting devices for foot traffic measurement
- ShopperTrak Analytics: centralised performance dashboard (labour optimisation, marketing effectiveness, merchandise planning)
- Heatmaps with shopper demographic and movement patterns
- Shopper Journey solution: multi-technology (people counting, Wi-Fi, mobile, video) path analytics
- Employee Exclusion solution: removes staff from traffic counts for accurate conversion rates
- Market Intelligence: benchmarking by geography and retail sub-vertical
- In-Store Flash Traffic API: 15-minute interval foot traffic counts
- Enterprise Daily Traffic API: previous-day store-level counts

**Differentiating features**
- Re-ID-enabled technology (Orbit AI) for journey continuity without biometric identification
- Managed Service model: ShopperTrak operates the counting infrastructure on behalf of clients
- Market benchmarking across thousands of retail locations in the ShopperTrak network
- Flash Traffic API for near-real-time POS and workforce integration

**UX patterns**
- Enterprise portal (portal.shoppertrak.com) with guided reporting
- Analytics surfaced for property investors and retail management, not just store managers

**Integration points**
- Flash Traffic API: 15-minute traffic counts → POS, mobile apps, enterprise reporting, workforce management
- Daily Traffic API: historical store data export

**Known gaps**
- Enterprise pricing and managed-service model excludes SMB retailers
- Re-ID technology is under regulatory scrutiny in the EU (biometric-adjacent)
- API access limited to Managed Service clients

**Licence / IP notes**
- Commercial / proprietary (Johnson Controls subsidiary). Re-ID white paper published but technology is proprietary.

---

### Dor Technologies

**Core features**
- Thermal-sensing, battery-operated people counter (peel-and-stick, no network dependency)
- Cloud dashboard: multi-store foot traffic, conversion rate, revenue correlation
- Date-range and seasonality comparisons
- Real-time weather data integration to explain traffic anomalies
- POS integration: automatic conversion rate calculation (Square, Lightspeed)
- Shopify integration for unified online + in-store view
- CSV export and API for BI tool connectivity
- Customisable reporting

**Differentiating features**
- Zero-network hardware: operates offline, no IT infrastructure required
- Fastest hardware setup: sensor installs in minutes without network configuration
- Lowest barrier to entry in the category; relevant for independent and SMB retailers
- Weather data overlay is built-in, not an add-on

**UX patterns**
- Simple mobile app (iOS) alongside web dashboard
- Designed for non-technical store managers
- Progressive disclosure: basic metrics by default, drill-down available

**Integration points**
- Square POS (native integration)
- Lightspeed POS (native integration)
- Shopify
- Generic CSV export + REST API for custom BI

**Known gaps**
- No heatmap or zone-level analytics (single-entrance counting only)
- No demographic analytics
- No staff scheduling integration
- Limited multi-store benchmarking vs enterprise peers
- Funded in 2017 ($3.8M seed); limited recent product news

**Licence / IP notes**
- Commercial SaaS + hardware. No open-source components identified.

---

### Placer.ai

**Core features**
- Location intelligence using aggregated mobile device data (GPS/location signals)
- Foot traffic analytics by day, week, month, year with trend lines
- Demographic breakdowns (age, income, psychographics) from mobile panels
- Trade area mapping: coordinates and polygon data for catchment areas
- Competitive benchmarking: compare performance across competing locations
- Site selection analytics: planned development, business counts, crime, climate overlays
- Sales estimation based on traffic and demographic modelling
- REST API with daily data export for BI integration

**Differentiating features**
- Mobile-panel methodology: works without any hardware at the location
- 92–96% accuracy claimed for foot traffic using aggregated device data
- Civic and CPG applications beyond retail
- Widest geographic coverage of any vendor surveyed

**UX patterns**
- Web analytics platform with map-based exploration UI
- Polygon-based store definition; users draw trade areas
- API-first for enterprise; dashboard for analysts

**Integration points**
- REST API (docs.placer.ai) with daily auto-exports
- Integration with internal BI tools, CRM, and custom prediction models
- Supports combining foot traffic data with POS data for conversion rate calculation

**Known gaps**
- No in-store analytics (zone heatmaps, dwell time, queue management) — external only
- Mobile panel data has inherent biases (smartphone ownership, location permissions)
- GDPR applicability to mobile-panel data is an ongoing regulatory question
- No staff scheduling or basket analysis capabilities

**Licence / IP notes**
- Commercial SaaS. Proprietary mobile panel data. No open-source components.

---

### Aislelabs

**Core features**
- WiFi and Bluetooth-based in-store analytics (leverages existing WiFi infrastructure)
- Zone-level dwell time and path analysis
- Heatmaps: visitor movement and engagement zones
- Occupancy monitoring
- Cross-visit and return visit metrics
- WiFi captive portal with email acquisition and verification
- Location-triggered email marketing and loyalty programme integration
- SMS marketing and authentication
- Advanced Visualisation API

**Differentiating features**
- WiFi-first approach: no additional hardware if WiFi is already deployed
- Combined people-counting and WiFi analytics via partner integrations (e.g. Axper)
- Strong retail property / shopping centre positioning (tenant benchmarking)
- Visitor email acquisition built into the analytics workflow

**UX patterns**
- Dashboard with zone visualisation and funnel metrics
- Marketing tools co-located with analytics (combined CRM + analytics use case)

**Integration points**
- Axper and other people-counting platforms (hardware-agnostic WiFi layer)
- Advanced Visualisation API for external dashboard embedding
- Loyalty programme integrations

**Known gaps**
- WiFi-based tracking has lower spatial resolution than camera-based approaches
- Requires guests to be on WiFi or have WiFi enabled on device for tracking
- POS and basket analysis not part of the core offering
- Staff scheduling not directly supported

**Licence / IP notes**
- Commercial SaaS. No open-source components identified.

---

### Trakwell.ai

**Core features**
- AI computer-vision people counting (99%+ accuracy via proprietary ML algorithms)
- 69+ real-time charts and reports across 75 retail metrics
- Conversion rate tracking linked to POS and staff data
- Staff performance measurement (service levels, follow-up effectiveness)
- Retail Equation Simulator: scenario modelling for traffic, staffing, conversion, service
- Staff scheduling aligned to peak traffic predictions
- Inventory insights and operational metrics

**Differentiating features**
- Retail Equation Simulator: closed-loop simulation of staffing decisions on conversion outcomes
- Focus on staff behaviour as a conversion driver (not just traffic measurement)
- 75-metric coverage is unusually broad for a single platform
- Direct POS + staff data integration as core architecture (not an add-on)

**UX patterns**
- Dashboard targeted at store managers and regional managers
- Simulation/scenario planning mode for managers without data science skills
- Alert-based notifications for service level thresholds

**Integration points**
- POS systems (direct integration for conversion rate calculation)
- Staff management systems (scheduling and performance data)

**Known gaps**
- No geographic / market benchmarking (location intelligence)
- Limited publicly available API documentation
- Smaller customer base and less established brand than RetailNext or Sensormatic

**Licence / IP notes**
- Commercial SaaS. No open-source components identified.

---

### Lightspeed Retail Analytics

**Core features**
- 50+ built-in POS reports (sales, inventory, employee performance, basket metrics)
- Smart dashboards: period-over-period (PoP) analysis with multi-view visualisation
- Basket analysis: basket size, basket value, items per transaction, return counts
- Employee performance analytics: which staff drive highest basket values
- Sell-through rate and stockout analysis
- Schedulable and customisable report exports
- Lightspeed Insights: visual, daily/weekly/monthly reporting for non-technical managers
- Foot traffic integration via Dor native integration

**Differentiating features**
- Native POS integration means basket-level data is always complete and accurate
- Staff performance linked to basket metrics for coaching purposes
- Dor integration closes the loop between foot traffic and POS transaction data

**UX patterns**
- Dashboard embedded in POS management UI; managers do not need a separate tool
- Scheduled report delivery (email) for managers who do not log into dashboards regularly

**Integration points**
- Dor people counter (native)
- Shopify (for unified online + offline view)
- GraphQL Admin API and REST API for external integrations
- Webhook support for real-time event-driven integrations

**Known gaps**
- Foot traffic analytics dependent on Dor hardware purchase
- No zone-level heatmaps or in-store path analytics
- Basket analysis is POS-side only; no product co-purchase association rule mining
- No multi-store location benchmarking against external competitors

**Licence / IP notes**
- Commercial SaaS. Proprietary POS platform. No open-source components identified.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Real-time and historical foot traffic counts per store, per day, per hour
- Conversion rate calculation (visitors ÷ transactions)
- Multi-store comparison and benchmarking dashboard
- Period-over-period trend views (day, week, month, year)
- CSV/data export for offline analysis
- Role-based access (store manager vs. regional manager vs. head office)

### Differentiating Features
- In-store zone heatmaps and dwell time by product area
- Staff exclusion from people counts for true conversion rate accuracy
- Weather, promotions, and event data overlay on traffic trends
- Shopper journey path tracking (entry to exit)
- Retail Equation Simulator / staffing scenario modelling
- Mobile-panel location intelligence for trade area and site selection
- AI-driven anomaly alerts (unexpected traffic spikes, queue thresholds)
- Re-ID / continuity tracking for multi-zone journey stitching

### Underserved Areas / Opportunities
- **Basket analysis linked to traffic**: knowing which product adjacencies correlate with higher basket value during specific traffic conditions is not natively supported by any single tool
- **AI-generated scheduling recommendations**: most tools surface staffing insights as dashboards; few generate recommended rosters automatically
- **Natural language querying**: store managers cannot ask questions in plain English; they must navigate fixed dashboards
- **Causal insight, not just correlation**: tools show that high traffic → higher sales, but do not explain why conversion drops during certain periods
- **SMB-accessible full stack**: the combination of traffic + basket + scheduling in one affordable package does not exist below enterprise price points
- **Privacy-native architecture for video**: GDPR and EU AI Act compliance is largely left to the buyer; few tools offer built-in consent and anonymisation workflows
- **Offline-first analytics**: most platforms require cloud connectivity; edge-processed analytics for poor-connectivity stores is rare

### AI-Augmentation Candidates
- Staff scheduling: AI can generate optimal rosters from traffic forecasts without manual intervention
- Anomaly detection: AI can identify unusual traffic patterns (weather, nearby events, competitor activity) and proactively alert managers
- Natural language interface: managers asking "Why was last Saturday's conversion rate 3% below average?" without writing SQL
- Market basket association rules: AI-driven product co-purchase analysis surfaced as plain-language layout recommendations
- Predictive traffic forecasting: machine learning on historical patterns, weather, and promotional data to predict next week's hourly traffic

---

## Legal & IP Summary

No open-source-licensed tools were identified among the leading retail analytics platforms surveyed. All major solutions (RetailNext, V-Count, Sensormatic ShopperTrak, Dor, Placer.ai, Aislelabs, Trakwell.ai, Lightspeed) are proprietary commercial SaaS products. RetailNext holds patent-pending IP on its Aurora sensor algorithm. Sensormatic holds proprietary Re-ID technology (Orbit AI). No patent conflicts were identified for open-source implementations of foot traffic counting, basket analysis, or staff scheduling optimisation using standard algorithms (Apriori, FP-Growth, Prophet). GDPR and EU AI Act compliance requirements impose design constraints on any system capturing in-store video or demographic data — these are regulatory obligations, not IP barriers.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Foot traffic ingestion from at least one people-counting sensor (via API or CSV import)
- POS transaction data ingestion (Square, Lightspeed, Shopify APIs)
- Conversion rate dashboard per store, per hour/day/week
- Multi-store benchmarking: key operational metrics on a comparable basis
- Staff scheduling recommendations generated from traffic forecasts
- Privacy-by-design data pipeline: no identifiable personal data stored or transmitted

**Should-have (v1.1)**
- Market basket analysis: product co-purchase association rules with store layout implications
- Zone-level heatmap integration (when sensor hardware supports it)
- Natural language query interface for store managers
- Weather and promotions data overlay on traffic trends
- AI anomaly detection and alerting (traffic spikes, conversion drops)

**Nice-to-have (backlog)**
- Retail Equation Simulator: scenario modelling of staffing vs. conversion outcomes
- Trade area and site selection analytics (using mobile-panel data via Placer.ai API)
- Shopify online + in-store unified view for omnichannel retailers
- Automated scheduled report delivery (email digest for non-dashboard users)
- GDPR consent and anonymisation workflow builder
