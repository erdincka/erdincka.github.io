---
layout: post
title: "Deciding what crosses the link: an edge-to-core pattern on HPE Data Fabric"
date: 2026-09-06
tags: [data-fabric, edge, streaming, storage]
description: >-
  A field team on a satellite link cannot pull down everything headquarters has.
  That is not a bandwidth problem, it is a decision problem — and it is a good test
  of whether one platform really beats an assembled stack.
image: /assets/img/satellite-interface.png
---

> **Before you read this.** This is an engineering demo, not a production reference
> architecture. Every figure below was measured against a live HPE Data Fabric 8.1
> cluster rather than estimated, but it is a single lab cluster with both sites on it —
> the numbers show that the mechanisms work, not what they do at fleet scale. The
> repository ships the provisioning, the verification and the failure modes, so the
> claims can be checked rather than taken on trust.

A field team working off a satellite link cannot pull down everything headquarters has.
That sounds like a bandwidth problem, and most designs treat it as one — compress harder,
sync less often, buy a bigger pipe. It isn't. It's a **decision** problem. The team needs
to know what *exists* so they can choose the few things worth spending the link on.

[Satellite Demo](https://github.com/erdincka/satellite) is a working model of that
pattern, built on [HPE Data Fabric](https://www.hpe.com/us/en/products/software/data-fabric-software.html).
An HQ site ingests imagery, catalogues it, and continuously broadcasts lightweight
*descriptions* to every edge site. The edge browses those descriptions and requests the
handful it actually wants. Only then does the imagery itself move. A vision model at the
edge can describe what arrived, so an operator gets an answer without opening every file.

The pattern generalises well beyond satellite imagery — disaster response, maritime
operations, remote industrial sites, defence — anywhere the link is expensive,
intermittent, or both.

![Headquarters on the left, the edge site on the right, and the live Data Fabric link between them](/assets/img/satellite-interface.png)

<figure>
  <video controls preload="metadata" poster="/assets/img/satellite-interface.png">
    <source src="/assets/video/satellite-demo.mp4" type="video/mp4">
    Your browser does not support embedded video.
    <a href="/assets/video/satellite-demo.mp4">Download the walkthrough</a> instead.
  </video>
  <figcaption>Ninety-seven seconds against the live cluster: an asset requested at the
  edge, the mirror pulled deliberately, then the link cut and restored while the backlog
  builds and drains.</figcaption>
</figure>

## The architecture in one idea

**Separate the metadata plane from the data plane, and give them different transports and
different cadences.**

Descriptions are small, and they travel *continuously* over a replicated stream. Imagery
is large, and it travels *only when asked for*, by volume mirroring the edge triggers
itself. Two planes, two mechanisms, two economics.

That separation is what makes the system usable on a bad link. The catalogue stays
current for pennies. The expensive transfer happens under human control, at a moment
someone has decided is worth it.

## One pipeline, four data models

Here is what makes this interesting as a platform story. That single pipeline touches
four different kinds of data, and each wants a different interface:

| In the pipeline | Data model | Interface used |
|---|---|---|
| Catalogue broadcast, asset requests | Event stream | Kafka API |
| Satellite imagery | Files | POSIX, over NFS |
| Asset metadata and AI narration | Table | Iceberg |
| The Iceberg warehouse itself | Objects | S3 |
| Volumes, streams, topics, mirrors | Administration | REST |

**None of that required a second system.** No message broker to deploy, no object store
to stand up, no separate catalog service, no file-transfer scheduler. One cluster, one
namespace, one identity — addressed through whichever standard interface each part of the
pipeline naturally speaks.

That is what multi-modal and multi-protocol mean in practice: the platform is not a
file system with an object gateway bolted on, or a message queue with a storage tier
attached. Files, objects, streams and tables are first-class, they live in the same
namespace, and they are governed by the same security model.

The practical consequence is that data does not have to move between systems to become
usable by a different consumer. An operator's imagery lands on a volume via POSIX; the
catalogue that indexes it is an Iceberg table on S3; the notification that it exists is a
stream event. Same cluster, same volumes, same credentials — no ETL between tiers whose
only purpose is to satisfy the next tool's preferred protocol.

## What you would otherwise assemble

It is worth being concrete about the alternative, because "one platform" is easy to say
and hard to evaluate. To build this pattern on general-purpose components you would need,
at minimum:

| Requirement | Assembled stack | HPE Data Fabric |
|---|---|---|
| Events between sites | Kafka plus MirrorMaker | Stream replication, multi-master |
| Bulk file transfer | rsync or a sync service, plus a scheduler | Volume mirroring, on demand or scheduled |
| Object storage | MinIO or Ceph, plus its own replication | S3 on the same volumes |
| Table storage | A warehouse and a catalog service | Iceberg on the same object store |
| File access at the edge | NFS server, separately deployed | Native |
| Identity across all of it | Per-system auth, federated somehow | One cluster identity, one ticket |
| Provisioning | Automation against several APIs | One REST API |

Every row in that middle column is a component to deploy, secure, monitor, patch, and
capacity-plan — at **every site**, including the ones with no staff and a satellite link.

And each brings its own disconnection semantics. MirrorMaker's lag behaviour is not
rsync's retry behaviour, which is not object replication's queueing behaviour. When the
link drops, you are reasoning about three or four independent recovery models at once,
and the interactions between them are where the incidents come from.

On Data Fabric, disconnection is handled the same way for everything, because it is the
platform's concern rather than each component's.

![Published at HQ against received at the edge, through a cut and a restore](/assets/img/satellite-backlog.png)

Cutting the link left HQ publishing normally while the edge stayed frozen and the backlog
grew to 21 messages; restoring it drained to zero within seconds. Imagery behaved
consistently: a requested asset simply sat staged in HQ's outbound volume until the edge
chose to mirror it. Nothing about that is simulated, and none of it is application code.

These are ordinary Data Fabric objects, visible and managed like any other — the edge's
imagery volume is a mirror, and the platform knows it:

![The demo's five volumes in the Data Fabric control system, with satellite-edge-assets typed as a mirror](/assets/img/satellite-mcs-volumes.png)

## What still needs designing

The platform removes the infrastructure problem. It does not remove the design problem,
and three decisions mattered more than any other.

### Model the link as a state, not an error

The instinct is to treat disconnection as an exception. It isn't — for an edge deployment
it is a normal operating mode, and often a *scheduled* one. Designing for three explicit
states (connected, scheduled, cut) produced a much better system than designing for
"connected, with error handling". The platform supports this directly: pausing and
resuming replication is a supported operation, and mirrors are on-demand by nature.

The corollary is to show the link's real state, read from the cluster, rather than
inferring it from whether your own messages happen to be flowing.

### Audit what you replicate

HQ's internal service hand-off runs on its own unreplicated stream. It would have been
one line of configuration to put it on the replicated pair — and it would have pushed
HQ's private chatter down the very link the design exists to conserve.

Because replication is so easy to enable, it is equally easy to enable by accident. On a
constrained link that is a real and recurring cost. Decide deliberately what crosses.

### Name objects by site, not by cluster

Each site owns its own volumes, streams and buckets, distinguished by name rather than by
which cluster they happen to live on. The system then behaves identically whether both
sites share one cluster or sit on two with a trust relationship. One is not a degraded
version of the other, and moving from one to two is a configuration change rather than a
redesign.

## From two sites to a thousand

This demo is a baseline, not a blueprint. A real deployment has tens or thousands of edge
sites, and at that scale the platform features that matter are the ones you never see in
a two-site demo. It is worth naming them, because they are the reason this pattern scales
without becoming a bespoke integration project per location.

### One namespace across many fabrics

Data Fabric's **global namespace** aggregates remote and disparate data sources so that
multiple fabrics can be viewed and operated as a single logical, local fabric. An
application addressing data at another site does not need to know it is remote.

Naming then becomes a fleet convention rather than a per-deployment choice: every cluster
in the namespace needs a unique name, and `mapr-clusters.conf` must carry the same
cluster configuration and naming across nodes and clients. The two-site convention in
this demo scales only if it is designed as a scheme — region, role, site identifier —
before the fleet exists rather than after.

### Authorization that spans clusters

For clusters to communicate at all, a secure **trust relationship** must exist between
them. That trust is what permits remote commands, remote replicas and mirrors, and NFS
access to another cluster — the exact operations this pattern depends on.

Setting it up is a defined procedure rather than a bespoke integration:
`configure-crosscluster.sh` generates a cross-cluster ticket, copies it to the other
cluster's CLDB node, merges it into that cluster's `maprserverticket` and distributes it;
`manageSSLKeys.sh` merges trust stores when a client must reach several clusters.
Identity remains the platform's, not each component's — which is the point. In an
assembled stack, cross-site authorization means federating several independent identity
systems and keeping them consistent across sites you cannot reach.

### The link itself is managed

Data Fabric treats a constrained link as something to manage rather than saturate.

**Mirroring throttles itself.** The sending server continuously measures round-trip time
and restricts mirror traffic to roughly 30% of available bandwidth, backing off when
other traffic needs it. Throttling can be disabled where a maintenance window justifies
full speed.

**Replication is compressed and can be encrypted.** Data Fabric applies network
compression — `lz4` by default — and traffic between secure clusters can be encrypted on
the wire. In this demo the replica record reports exactly that: `networkcompression:
lz4`, `networkencryption: false` on a lab cluster where encryption was not required.

**Mirrors run on schedules with priorities.** Schedules carry pre-assigned meanings —
critical, important, normal — so a fleet can express that some sites sync hourly and
others when someone asks, without writing a scheduler.

### Operations and observability at fleet scale

Data Fabric Monitoring collects metrics and logs for nodes, services and jobs, which is
what makes a fleet observable centrally rather than site by site. Note that monitoring
components are not installed on client or edge nodes — an edge site reports through its
own cluster, so the observability design follows the fabric topology.

For genuinely small sites there is **HPE Data Fabric Edge**: a small-footprint
edition running on commodity hardware in three- to five-node configurations, with the
full capability set — files, tables and streams, plus snapshots, mirroring, replication
and compression. An edge site is a small fabric, not a cut-down client, which is why the
same design works at both ends.

## Trying it

The demo runs as a single container against your own cluster. Both sites can point at the
same cluster — they stay separated by volume, stream and bucket names rather than by
geography — so you do not need two clusters to see the pattern work:

```
git clone https://github.com/erdincka/satellite
cd satellite
docker compose up -d
```

It starts with no configuration and is pointed at a cluster from the interface, which is
the easiest way to try it. The
[README](https://github.com/erdincka/satellite/blob/main/README.md) covers the
prerequisites that actually matter — a reachable CLDB, apiserver, S3 and NFS, a client
version matching the cluster, and `CAP_SYS_ADMIN` on the container because imagery is
reached over an NFS mount.

---

## The takeaway

The interesting claim isn't that Data Fabric can move data between sites. Plenty of
things move data. It is that a pipeline spanning streams, files, tables and objects —
across sites, over a link that comes and goes — can be built on **one platform, through
standard interfaces**, instead of assembled from components that each need their own
deployment, security domain, replication mechanism and failure model.

That leaves the application with the one job it should have: deciding what is worth
sending, and when. The fabric handles delivery, queueing, resumption and consistency
underneath.

The application decides. The fabric delivers.
