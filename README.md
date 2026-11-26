# 6WIND VSR License: What Base and Application Tiers Cover, Why Activation Fails, and How a Maintenance End Date Caps Your Upgrades

A 6WIND VSR license is two things sold together: a base license that sets throughput, and an application license that sets IPsec tunnel count and concurrent connections. The official ceilings are 100 Mbps to 200 Gbps of throughput, 100 to 100K IPsec tunnels, and 1M to 100M concurrent connections. 6WIND does not publish a price list for these; the License Fee is whatever the accepted Purchase Order says, and it is subject to annual revision.

That combination model explains most of the confusion around this product. The rest of this article covers how activation actually works, what happens at an isolated site with no route to the license server, the day-one configuration trap that leaves engineers unable to activate anything, why a perpetual license still stops you from running newer releases, and how to pick a capacity tier before you ask for a quote.

> **Quick version**
> - Fix three numbers before you contact anyone: throughput tier, IPsec tunnel tier, concurrent connection tier. Each product in the VSR family (Security Gateway, CG-NAT Router, Border Router, Provider Edge Router, Cell Site Router, CPE Router) is licensed separately.
> - A license key is valid for 30 days at a time. A health check extends it by another 30 days on every successful check, which means any VSR without a path to the license server eventually stops working unless it was activated offline.
> - A perpetual license covers only releases published before your maintenance end date. Renewing maintenance is what moves that date.
>
> 👉 [See 6WIND VSR licensing options and request pricing](https://sixtynet.net/aff.php?aff=39)

## What the license key actually consists of

A license key unlocks features and capacities, and it is defined as a base license plus an application license. The base license controls throughput; the application license controls the number of IPsec tunnels and the number of concurrent connections. Different levels exist for each product in the family, so a Border Router key and a Security Gateway key are not interchangeable even at the same throughput.

| Component | What it controls | Official capacity range | Pricing |
| --- | --- | --- | --- |
| Base License | Throughput | 100 Mbps to 200 Gbps | [Request a quote](https://sixtynet.net/aff.php?aff=39) |
| Application License | Number of IPsec tunnels | 100 to 100K tunnels | [Request a quote](https://sixtynet.net/aff.php?aff=39) |
| Application License | Concurrent connections | 1M to 100M CC | [Request a quote](https://sixtynet.net/aff.php?aff=39) |

Two things follow from that table. First, the tunnel and connection ranges are wide enough that the top tiers are carrier-scale numbers, not something a branch office needs. Second, because throughput is a separate axis, you can end up buying a 200 Gbps base license with only 100 tunnels, or a modest throughput tier with a very large connection table. The dimension that matters depends on the role: a security gateway lives on tunnel count, a CG-NAT or BNG deployment lives on concurrent connections, and a border router lives on throughput.

Documentation drift is worth knowing about here. Older VSR 3.4.8 documentation listed the application ranges as 100 to 40K tunnels and 1M to 30M concurrent connections. The current 3.7.18 documentation states 100K tunnels and 100M connections. If a quote or an internal design document references 40K or 30M, it was probably built from the older figures, and you should confirm which one the order maps to.

## Activation, the 30-day lease, and what your router reports back

Activating a license requires the VSR to reach the license key server, which means DNS and an internet route have to be configured first. Each activation consumes one activation on the key server, and the license stays valid only while it is enabled server-side with activations remaining.

A license key is valid for 30 days. A health check mechanism contacts the license server periodically and automatically, and a successful check resets the validity period for another 30 days. This is not a one-time activation check; it is an ongoing lease. The license agreement states the same thing from the other direction: if the license verification process is not executed periodically, the software stops working.

Activation and health checks report device information upward: the instance's IP address, version data (OS, version, build), and environment details (CPU, memory, hostname). If your security review covers telemetry, that is the full list to review.

Invalidation has two different timings, and mixing them up causes outages. Removing the license key from the configuration invalidates it immediately. Deactivating it on the server takes effect at the next health check. Either way, features and capacity become restricted afterward.

## Offline activation for sites that cannot reach the license server

Some specific licenses can be activated offline through the Licensing End-User Portal on the 6WIND Customer Zone, using a web page instead of a connection between the VSR and the license server. This is the path for isolated points of presence, lab racks behind a firewall with no egress, and any deployment where the management plane has no internet route.

6WIND's documentation gives the command shapes for this workflow (the examples come from the Turbo Router 3.x documentation, so treat the syntax as indicative rather than as verified VSR 3.x grammar):

1. On the machine that will consume the token, request an activation certificate: `cmd license certificate request-activation serial <serial>`. This only runs while the license is disabled in the configuration.
2. Log in to the Licensing End-User Portal, open the `OfflineActivation` tab, paste the certificate, and click `Activate`.
3. Import the resulting certificate back: `cmd license certificate import content <certificate-from-webpage> serial <serial>`. The import command also accepts ftp and http URLs, and file contents can be pasted straight into the console.
4. Configure `system license offline-certificate serial <serial>`, commit, and exit edit mode. Activation only takes effect after you leave edit mode.

That last point applies to the online path as well: `system license online serial <serial>` followed by commit does nothing until you exit edit mode.

Offline activation also makes it easy to strand activations. A token stays consumed until you free it: run `cmd license certificate request-deactivation serial <serial>`, paste the resulting certificate into the portal, and click `Deactivate`. Do this when you decommission a node or rebuild a lab box, otherwise your remaining activation count quietly shrinks.

Whether your particular license supports offline activation is a per-order question. Ask before the PO is signed, not after the hardware is racked.

👉 [Ask about offline activation and activation counts for your order](https://sixtynet.net/aff.php?aff=39)

## The catch-22: you cannot use BGP or IPsec to reach the license server

If the path from the VSR to the license key server depends on a feature that the license itself enables, such as BGP or IPsec, and you do not yet have a valid license, the activation cannot complete. 6WIND's own documentation calls this out and recommends a simple day-one configuration using static routing to install the license key first, then moving on to the production design.

This trap catches designs where the VSR is meant to sit on an IPsec overlay or learn its default route over BGP from day one. Build the bootstrap config with a static default route and a working resolver, bring the license up, verify the health check succeeds, and only then push the real routing and tunnel configuration. It also means your deployment runbook needs a temporary management path that exists before the license does.

## Maintenance end date decides which releases you can run

Each license key carries a maintenance end date, corresponding to the end of the maintenance period after the first delivery of the VSR. Renewing the maintenance agreement updates that date. The key is not valid for VSR releases published after the maintenance end date.

The license agreement's version definitions make the practical effect clear. A maintenance release changes the z digit of x.y.z, a minor release changes the y digit, and a major release changes the x digit. Updates means maintenance plus minor releases; Upgrades means major versions. So a perpetual license with lapsed maintenance does not block you from running the software forever, it freezes you at the last release published before your maintenance end date. If you are planning a platform refresh eighteen months out, that date, not the license expiry, is what determines whether you can take the newer train.

Delivery timing matters for planning: 6WIND makes the Master Copy and documentation available for download and posts the license keys on the License Key Server within five business days of receiving an accepted Purchase Order, and delivery is deemed to occur on the date you are notified that both are available.

👉 [Check what a current VSR license includes on the maintenance side](https://sixtynet.net/aff.php?aff=39)

## Contract terms that constrain the architecture

The perpetual license agreement contains several limits that never appear on a product page but shape how you deploy.

License keys cannot be shared outside the end-user organization, and the software cannot be used beyond its capacity. The consequence of exceeding capacity is spelled out: usage will be limited, the software may not function, and maintenance services may be interrupted. Exceeding capacity is also listed as a material breach that can trigger termination if not remedied within thirty days of notice.

High availability is licensed by role. A Hot Standby or Warm Standby instance may be paired with an Active instance; using a standby instance as an active one requires a new Purchase Order to convert the license. If you plan to promote a standby node permanently after a site consolidation, budget for that conversion.

Commercially, the License Fee is set per unit in the Purchase Order and is subject to annual pricing revision. Late payment accrues interest at 1.50% per month or the highest rate permitted by law, whichever is lower, and 6WIND can suspend maintenance and engineering consulting until the balance clears.

## Sizing the throughput tier against the platform

On Equinix's certified VNF listing, 6WIND VSR is a BYOL product: you buy and manage the license yourself from 6WIND or a partner, and BYOL licenses are eligible for 6WIND vSR support. The published规格 pairs cores with memory: 2 cores with 8 GB, 4 cores with 16 GB, 8 cores with 32 GB. Virtual data interfaces are 10 by default and 24 maximum, deployment options are single or redundant, access is web console and SSH, and no CLI commands are restricted.

That mapping is the practical constraint on the base license. Buying a throughput tier that your core count and memory cannot feed wastes money; buying a tier below your traffic profile gets you throttled. Match the base license to the instance size you are actually deploying, then size the application license against the role.

## Picking a tier: three profiles

Small PoP or branch edge doing site-to-site encryption: a low throughput base license plus the 100-tunnel application tier covers most of these, and the connection table is irrelevant. Do not buy tunnel headroom you have no peers for.

Hub or VPN concentrator aggregating many branches: calculate peak concurrent tunnels first, including standby peers and any temporary migrations, then pick the tunnel tier. Throughput still has to cover the aggregate, but tunnel count is the number that gets exceeded silently.

CG-NAT, BNG, or any subscriber-facing function: concurrent connections is the binding constraint. Subscriber counts times sessions per subscriber, with headroom for NAT oversubscription, is the number to bring to the quote.

Three things to confirm before the PO goes out: the maintenance end date attached to the key and what renewal costs, whether your sites need offline activation and how many activations the order includes, and whether your HA pair is quoted as standby or as two active instances.

👉 [Start a 6WIND VSR license quote with those three numbers in hand](https://sixtynet.net/aff.php?aff=39)
