# Topology-Aware Load Balancer — Change Log

Branch: `topo-aware`  
Base commit: `67b1418` (master HEAD at branch creation)

---

## Motivation

The default OVN-Kubernetes load balancer distributes service traffic uniformly
across all healthy endpoints regardless of node proximity. In a
densely-connected microservice application like Online Boutique, this means
every ClusterIP call crosses the east-west fabric even when a local replica
exists, adding per-hop overlay encapsulation and inter-node bandwidth cost.

This set of changes implements a **topology-aware load balancer** that gives
each node a per-node OVN LB whose backend pool follows a 3-tier locality
hierarchy, progressively widening scope until healthy backends are found:

```
Tier 1 – Node-local:   same node as the calling pod (zero east-west hops)
Tier 2 – Zone-local:   same topology.kubernetes.io/zone  (intra-zone fabric)
Tier 3 – Region-local: same topology.kubernetes.io/region (intra-region, cross-zone)
Tier 4 – Cluster-wide: all healthy endpoints (full fallback)
```

Each tier applies a proportionality guard: if the tier holds less than 50% of
its expected share of cluster endpoints (mirroring the Kubernetes
`TopologyAwareHints` check), it is skipped and the next tier is tried.  This
prevents a single underprovisioned pod from becoming a hotspot while still
allowing HPA to observe representative CPU load.

The feature is **off by default** and is activated via the
`--enable-topology-aware-lb` flag, guarded additionally by a per-service
opt-in annotation (`service.kubernetes.io/topology-mode: Auto`).

---

## Changed Files

### 1. `go-controller/pkg/config/config.go`

**What changed:** Added `EnableTopologyAwareLB` field to
`OVNKubernetesFeatureConfig` and the corresponding `--enable-topology-aware-lb`
CLI flag.

**Key additions:**

```go
// In OVNKubernetesFeatureConfig struct:
EnableTopologyAwareLB bool `gcfg:"enable-topology-aware-lb"`

// CLI flag registered in OVNKubernetesFeatureFlags:
&cli.BoolFlag{
    Name: "enable-topology-aware-lb",
    Usage: "Prefer same-zone endpoints for ClusterIP services annotated with " +
        "'service.kubernetes.io/topology-mode: Auto'. ...",
    Destination: &cliConfig.OVNKubernetesFeature.EnableTopologyAwareLB,
    Value:       OVNKubernetesFeature.EnableTopologyAwareLB,
},
```

**Why:** Following the existing pattern for feature gates (e.g.
`EnableServiceTemplateSupport`, `EnableEgressIP`), this flag guards the feature
at the cluster operator level so existing deployments are unaffected.

---

### 2. `go-controller/pkg/ovn/controller/services/node_tracker.go`

**What changed:**

1. Two new fields on `nodeInfo`:

   ```go
   topologyZone   string  // value of topology.kubernetes.io/zone node label
   topologyRegion string  // value of topology.kubernetes.io/region node label
   ```

2. `updateNodeInfo()` signature extended with `topologyZone, topologyRegion
   string` parameters; both are stored in the resulting `nodeInfo`.

3. `updateNode()` now reads the topology labels from the Node object and passes
   them to `updateNodeInfo()`:

   ```go
   node.Labels[corev1.LabelTopologyZone],
   node.Labels[corev1.LabelTopologyRegion],
   ```

4. The `UpdateFunc` handler now also triggers `updateNode()` when topology
   labels change, ensuring LBs are re-reconciled immediately if a node is
   re-labelled:

   ```go
   oldObj.Labels[corev1.LabelTopologyZone] != newObj.Labels[corev1.LabelTopologyZone] ||
   oldObj.Labels[corev1.LabelTopologyRegion] != newObj.Labels[corev1.LabelTopologyRegion]
   ```

**Why:** `nodeInfo` is the authoritative per-node fact sheet consumed by all LB
builders. Carrying topology labels here avoids extra node lookups deeper in the
call stack and ensures zone information is always consistent with the rest of
the cached node state.

---

### 3. `go-controller/pkg/ovn/controller/services/lb_config.go`

This is the core file for the topology-aware routing logic. Four distinct
changes were made.

#### 3a. New `preferLocalEndpoints` field on `lbConfig`

```go
// preferLocalEndpoints is true when the topology-aware LB feature is enabled
// and the Service carries "service.kubernetes.io/topology-mode: Auto".
preferLocalEndpoints bool
```

**Why:** `lbConfig` is the abstract intermediate representation shared between
`buildServiceLBConfigs()` and all three LB builders (`buildClusterLBs`,
`buildTemplateLBs`, `buildPerNodeLBs`). Adding the flag here keeps the
decision logic in one place and propagates it downstream without extra
parameters.

#### 3b. Detection logic in `buildServiceLBConfigs()`

Before the per-service-port loop, three changes were made:

1. **Pre-compute `externalTrafficLocal` and `internalTrafficLocal`** outside
   the loop (they are service-level, not port-level, so computing them once is
   correct and avoids redundant calls).

2. **Compute `preferLocalEndpoints`** — true when all of the following hold:
   - `config.OVNKubernetesFeature.EnableTopologyAwareLB` is set.
   - At least one node in `nodeInfos` carries a non-empty `topologyZone`.
   - The service annotation `service.kubernetes.io/topology-mode` equals `"Auto"`.
   - Neither ETP nor ITP is `Local` (those already have their own per-node
     mechanism and take precedence).

3. **Include `preferLocalEndpoints` in `needsLocalEndpoints`** so that
   `GetEndpointsForService` populates `portToNodeToEndpoints`; without this,
   the per-node endpoint data needed by `buildZoneEndpoints` would be empty.

Inside the per-port loop:

- `preferLocalEndpoints` is propagated into `clusterIPConfig.preferLocalEndpoints`.
- **Routing decision**: topology-aware ClusterIP configs are appended to
  `perNodeConfigs` (not `clusterConfigs`) because each node needs a different
  backend list.
- **Template LB guard**: `preferLocalEndpoints` is added to the condition that
  forces NodePort services to use per-node LBs instead of template LBs.

#### 3c. New `countTopologyZones()` / `countTopologyRegions()` helpers

```go
func countTopologyZones(nodes []nodeInfo) int
func countTopologyRegions(nodes []nodeInfo) int
```

Count the number of distinct non-empty topology zone/region labels across all
cluster nodes. Pre-computing these values once per service reconciliation
avoids repeating an O(N) pass inside every `buildZoneEndpoints` /
`buildRegionEndpoints` call. Without this, the per-node loop in
`buildPerNodeLBs` would trigger O(N × configs) redundant iterations over the
full node list.

#### 3d. New `buildZoneEndpoints()` / `buildRegionEndpoints()` helpers

```go
func buildZoneEndpoints(zone string, nodes []nodeInfo,
    nodeEndpoints map[string]util.LBEndpoints,
    port int32, clusterTotal, numZones int) util.LBEndpoints

func buildRegionEndpoints(region string, nodes []nodeInfo,
    nodeEndpoints map[string]util.LBEndpoints,
    port int32, clusterTotal, numRegions int) util.LBEndpoints
```

Each merges (with IP deduplication) the endpoint pools for every node in the
given zone/region. Both accept pre-computed `numZones`/`numRegions` counts
rather than recomputing them internally.

Return an empty `LBEndpoints` when:
- The label value is `""` (node/zone/region not labelled)
- No node in the scope has endpoints
- **Proportionality guard:** the scope holds fewer than 50% of its expected
  share of `clusterTotal` endpoints (`clusterTotal / numZones * 0.5`), preventing
  a single underprovisioned pod from becoming a hotspot. Mirrors the
  Kubernetes `TopologyAwareHints` proportionality check.

The empty-return sentinel causes `makeNodeSwitchTargetIPs` to fall through to
the next tier automatically.

#### 3e. Updated `makeNodeSwitchTargetIPs()`

Signature extended with `zoneEndpoints, regionEndpoints util.LBEndpoints`.
The full 4-tier priority order is now:

| Priority | Condition | Target IPs |
|---|---|---|
| 1 | `externalTrafficLocal \|\| internalTrafficLocal` | Node-local only (unchanged) |
| 2 | `preferLocalEndpoints && nodeEndpoints[node]` non-empty | **Node-local** (new tier 1) |
| 3 | `preferLocalEndpoints && len(zoneEndpoints) > 0` | **Zone-local** (new tier 2) |
| 4 | `preferLocalEndpoints && len(regionEndpoints) > 0` | **Region-local** (new tier 3) |
| 5 | default | Cluster-wide (unchanged) |

Each tier's non-emptiness is guaranteed by the proportionality guard in the
builder functions — an under-provisioned zone/region returns empty, causing
automatic fall-through to the wider scope.

**Known limitation — router path:** `makeNodeRouterTargetIPs` (used for the
GatewayRouter LB) is **not** topology-aware. Traffic that enters the cluster
via a GatewayRouter (e.g. NodePort, LoadBalancer-type services, or shared-GW
mode ingress) will use cluster-wide endpoints regardless of topology settings.
For the Online Boutique experiment this is acceptable — all service-to-service
calls use ClusterIP, which goes through the node switch LB (topology-aware).
Extending `makeNodeRouterTargetIPs` to apply zone filtering is left as future
work.

#### 3f. Updated `buildPerNodeLBs()`

Pre-computes `numZones` and `numRegions` **once** via the new helpers before
entering the per-node loop, then for each `(node, proto, config)` triple:

1. **Compute `zoneEps` and `regionEps`** via the updated builders when
   `cfg.preferLocalEndpoints`.
2. **Pass both** to the updated `makeNodeSwitchTargetIPs`.
3. **Add topo-aware switch rule branch**: when
   `cfg.preferLocalEndpoints && util.IsClusterIP(vip)`, switch rules use the
   zone-filtered targets instead of the cluster-wide `targets`.
4. **Mark LBs as `TopoAware`**: the `Opts.TopoAware` field is set to `true`
   on all LB objects created for a topo-aware config bucket.

The call site inside `buildTemplateLBs` (which never has
`preferLocalEndpoints=true`) passes `util.LBEndpoints{}` for both zone and
region endpoints to preserve existing behaviour with no data overhead.

---

### 4. `go-controller/pkg/ovn/controller/services/loadbalancer.go`

**What changed:**

1. New `TopoAware bool` field in `LBOpts`:

   ```go
   // TopoAware marks this LB as topology-aware (zone-local endpoints preferred).
   // When set, OVN uses (ip_src, ip_dst) selection fields so that ECMP hashing
   // is stable per source-IP pair, ensuring consistent routing within a zone.
   TopoAware bool
   ```

2. In `buildLB()`, added an `else if lb.Opts.TopoAware` branch that sets
   `selectionFields` to `[ip_src, ip_dst]`:

   ```go
   } else if lb.Opts.TopoAware {
       selectionFields = []nbdb.LoadBalancerSelectionFields{
           nbdb.LoadBalancerSelectionFieldsIPSrc,
           nbdb.LoadBalancerSelectionFieldsIPDst,
       }
   }
   ```

**Why:** OVN's default ECMP hashing uses the full 5-tuple. With a small
zone-local backend pool, 5-tuple hashing can cause uneven distribution.
Using `(ip_src, ip_dst)` ensures each client pod consistently hashes to the
same zone-local backend across reconnects, reducing cross-zone "spill" from
hash collisions.

Note: `neighbor_responder: none` was already set for **all** OVN LBs in the
existing `buildLB()` code; no change is needed there.

---

## Data Flow Summary

```
Node labelled with topology.kubernetes.io/zone=zone-a
        │
        ▼
nodeTracker.updateNode()
  → reads corev1.LabelTopologyZone
  → nodeInfo.topologyZone = "zone-a"
        │
        ▼
buildServiceLBConfigs()  [if --enable-topology-aware-lb AND annotation="Auto"]
  → preferLocalEndpoints = true
  → needsLocalEndpoints  = true  (per-node endpoint data requested)
  → clusterIPConfig.preferLocalEndpoints = true
  → clusterIPConfig routed to perNodeConfigs
        │
        ▼
buildPerNodeLBs()  [numZones/numRegions pre-computed once]
  → zoneEps   = buildZoneEndpoints("zone-a", nodes, nodeEndpoints, port, total, numZones)
  → regionEps = buildRegionEndpoints("region-1", nodes, nodeEndpoints, port, total, numRegions)
  → makeNodeSwitchTargetIPs(node, &cfg, zoneEps, regionEps)
       ├─ if nodeEndpoints[node] non-empty → returns node-local IPs  (tier 1)
       ├─ if zoneEps non-empty             → returns zone-local IPs   (tier 2)
       ├─ if regionEps non-empty           → returns region-local IPs (tier 3)
       └─ else                             → returns clusterEndpoints (tier 4 / fallback)
  → switch rule built with locality-filtered (or fallback) targets
  → router rule always uses cluster-wide endpoints (GatewayRouter path not topo-aware)
  → LB.Opts.TopoAware = true
        │
        ▼
buildLB()
  → selectionFields = [ip_src, ip_dst]  (stable ECMP within zone)
        │
        ▼
EnsureLBs() → single atomic OVN NB DB transaction
  → per-node LB with zone-local VIPs committed to northbound DB
```

---

## Behavioural Guarantees

| Scenario | Behaviour |
|---|---|
| Feature flag off | No change — existing cluster-wide LB behaviour |
| Feature flag on, service has no annotation | No change for that service |
| Feature flag on, node has local pod | **Tier 1**: per-node LB uses node-local endpoint (zero east-west hops) |
| Feature flag on, zone proportionally served | **Tier 2**: per-node LB uses zone-local endpoint pool |
| Feature flag on, region proportionally served | **Tier 3**: per-node LB uses region-local endpoint pool |
| Feature flag on, all locality tiers fail proportionality | **Tier 4**: automatic fall-back to cluster-wide backend pool |
| Zone/region has < 50% of proportional endpoint share | Tier skipped — falls to next tier (prevents single-pod hotspot) |
| ETP=Local or ITP=Local service | Existing per-node mechanism takes precedence; `preferLocalEndpoints` is not set |
| Node topology label changes | `nodeTracker.UpdateFunc` detects the change and triggers full service resync |

---

## How to Enable

```bash
# Start ovnkube-master / ovnkube-cluster-manager with:
--enable-topology-aware-lb

# Label your nodes:
kubectl label node node1 topology.kubernetes.io/zone=zone-a
kubectl label node node2 topology.kubernetes.io/zone=zone-b

# Opt in a service:
kubectl annotate svc my-service service.kubernetes.io/topology-mode=Auto
```
