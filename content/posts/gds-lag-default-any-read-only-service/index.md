---
title: "Your GDS read-only service has no staleness limit by default"
date: 2026-11-10T10:00:00-03:00
draft: false
categories: ["Oracle DBA"]
tags: ["oracle", "dba", "gds", "global data services", "active data guard", "data guard"]
summary: "GDS global services default to -lag ANY. What that means for a standby read-only service and how to set a real limit."
description: "GDS global services default to -lag ANY. What that means for a standby read-only service and how to set a real limit."
cover:
  image: "cover.png"
  alt: "No staleness limit by default: GDS -lag ANY"
---

Most GDS setups I see create the read-only service like this:

```bash
GDSCTL> add service -service sales_ro -gdspool sales -preferred_all -role PHYSICAL_STANDBY -failover_primary
```

It works. Reports go to the standby, and if the standby is gone the service moves to the primary. What the command does not say is how stale the data is allowed to be.

## The default is ANY

The `-lag` option of `add service` sets the maximum lag, in seconds, that a database may have and still offer the service. If you leave it out, the value is `ANY`: no upper limit.

So with the command above, a standby that is 40 minutes behind is still a valid target. GDS keeps routing to it. Nothing is broken from the GDS point of view, because you never told it what "too far behind" means.

## Setting a limit

```bash
GDSCTL> modify service -gdspool sales -service sales_ro -lag 30
GDSCTL> config service -service sales_ro -gdspool sales
```

With a limit in place, the documented behavior for Active Data Guard is:

- GDS routes to a standby whose lag is under the limit.
- If a standby goes over it, the service is relocated to another standby that is under the threshold.
- If no database qualifies, the service is shut down.
- When the lag comes back within the limit, GDS starts the service again.

## The part to decide before you set it

"The service is shut down" is the line that matters. A lag limit trades stale reads for a possible outage of the read-only service. That is often the right trade, but it should be a decision, not a surprise at 3 a.m. during a redo transport problem.

Two things I would test in a non-production pool before rolling this out:

1. Whether `-failover_primary` moves the service to the primary when every standby is over the limit. The documentation describes it as failover to the primary for a `PHYSICAL_STANDBY` service, but I did not find a statement covering the lag case specifically.
2. What happens to sessions already connected when the service stops on a lagging standby. The lag check governs where the service runs and where new connections go.

Pick the number with the application team. A package tracking screen can live with 10 seconds. A month-end report may be fine with `ANY`, and then the default was right all along.

## Technical validation

Confirmed in Oracle documentation ([GDSCTL add service, 26ai](https://docs.oracle.com/en/database/oracle/oracle-database/26/gsctl/add-service.html) and [GDS best practices, 19c MAA](https://docs.oracle.com/en/database/oracle/oracle-database/19/haovw/oracle-global-data-services-best-practices.html)):

- `-lag {lag_value | ANY}`, default `ANY`, value in seconds.
- A database whose lag exceeds the value cannot provide the service.
- Relocation, shutdown and automatic restart behavior for Active Data Guard.
- `-role` applies only to pools containing a Data Guard broker configuration.
- `-failover_primary` is valid with `-role PHYSICAL_STANDBY`.
- `-lag` is accepted by both `add service` and `modify service`.

Not confirmed, test in your environment:

- Interaction between a lag breach on all standbys and `-failover_primary`.
- Effect on existing sessions when the service stops due to lag.

Varies by environment:

- The right lag value depends on transport mode, network and what the application tolerates.
- Databases in a GDS configuration must be licensed for Active Data Guard or GoldenGate.
