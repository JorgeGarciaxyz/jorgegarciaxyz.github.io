---
layout: default
title: "Work"
css: index
---

# Work

I’ve worked as a Full-Stack Engineer for the past 10 years, primarily with Ruby on Rails
and many flavors of Frontend: Vanilla Rails, React, Ember, and Vue.

The vast majority of my career has been building SaaS products with remote teams across USA,
Mexico, and Canada.
# Selected Work

## Fleetio

Fleetio ([Member of the Rails Foundation](https://rubyonrails.org/foundation){:target="_blank"}) is a Fleet management software that helps
organizations track, analyze, and automate their fleet operations.

### Fleetio - Maintenance Shop Network Team
*January 2025 - currently*

### Fleetio - Integrations Team
*September 2021 - January 2025*

**Fleetio Ingest — Lead Engineer**

*High-throughput Ruby on Rails ETL service ingesting telematics and fuel-card data into Fleetio's core platform via Kafka.*

- Led and maintained **Fleetio Ingest**, a high-throughput **Ruby on Rails** ETL service built on the **Citadel architecture** to scale data ingestion independently from the core monolith — streaming transformed data from third-party **telematics and fuel-card APIs** into Fleetio via **Kafka** at ~20 MB/s with near-zero replication lag.
- Re-architected a critically **rate-limited pipeline** with a custom **Sidekiq Batches** solution, cutting end-to-end data lag from **~3 days to under 5 seconds** and increasing sustained throughput **3x+ (from ~3 MB/s to ~10 MB/s)** through Kafka batch-size and memory tuning.
- Standardized all **12 ingestion pipelines** onto a strict **ETL pattern with incremental synchronization**, eliminating redundant processing and preventing third-party rate-limit violations.
- Owned the service in production for **2.5 years**, sustaining **94%+ test coverage** and a **weekly dependency-update cadence** to keep the system secure, current, and reliably monitored.


