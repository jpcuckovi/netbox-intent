# netbox-intent

A network configured from a source of truth: NetBox holds what an SR Linux lab
should be, Jinja2 templates synced from this repository render each switch's
configuration, and a push applies it over gNMI with commit confirmed, verifies
the whole network against NetBox, and confirms only if everything holds.

> **Status, 2026-10-02: the pipeline runs end to end.** NetBox is seeded from
> this repository, renders all four switches from layered templates synced over
> git, and a single push brought the network from blank to converged: every
> OSPF adjacency full, every loopback reachable, a default route originated at
> the core and load-shared across both aggregation switches at the access
> layer. A dry run then reports zero differences on every switch, which is the
> baseline drift detection compares against.
>
> The safety net is proven against the running lab as of 2026-10-06. Two
> deliberately faulty pushes, one that the network checks fail and one that a
> switch rejects, were each cancelled on every switch that had committed, and
> the network returned to matching NetBox with nothing left behind.
>
> Drift is watched as it happens: a change made on a switch behind NetBox's
> back was reported within seconds, and reported as cleared once a push
> restored it. The watcher runs in the cluster beside the lab, from an image
> built from this repository.

## What this is for

The question it answers is how network elements are configured dynamically:
intent recorded once, rendered per device, applied as a transaction, and
checked afterwards against the same intent. Each stage is built by hand, so
each one can be explained.

```
seed/*.yml ──loader──▶ NetBox ◀──git sync── templates/srlinux/*.j2
                         │
                    render-config
                         ▼
              intended config per switch ──┐
                                           ├──▶ diff ──▶ push (gNMI, commit confirmed)
 running config per switch (gNMI Get) ─────┘                  │
                                                              ▼
                                       verify: read-back + OSPF + routes, from NetBox
                                                              │
                                              confirm all ◀───┴───▶ cancel all
```

The same diff serves three purposes: the plan before a push, the read-back
check during one, and drift detection at any time.

## The topology

```
               core1            originates the default route into OSPF
        e1-1 /       \ e1-2
      e1-1  /         \  e1-1
         agg1         agg2
      e1-2  \         /  e1-2
        e1-1 \       / e1-2
              access1           host gateway, ECMP toward both aggregation switches
                 | e1-3
               host1
```

Four Nokia SR Linux 26.7.2 switches and one netshoot host. Routed access with
single-area OSPF: every link between switches is a routed point-to-point link,
and BGP appears only at peering points, of which there are none yet.

| Use                   | Prefix          | Assignments                                                                    |
| --------------------- | --------------- | ------------------------------------------------------------------------------ |
| Loopbacks (`system0`) | `10.0.0.0/24`   | core1 `.1`, agg1 `.2`, agg2 `.3`, access1 `.4`, each a `/32` and the router ID |
| Point-to-point links  | `10.0.1.0/24`   | one `/31` per link, upstream end on the lower address                          |
| Host LAN              | `10.10.10.0/24` | access1 `ethernet-1/3` `.1`, host1 `.11`                                       |

The seed files are the authority for devices, cabling and addressing; the
containerlab topology repeats the cabling so containerlab can wire it, and a
test fails if the two disagree.

## Where it runs

The lab runs on a single-node k3s cluster under clabernetes, which runs each
containerlab node as a pod and builds the links between them. NetBox runs in
the same cluster, and so does the drift watcher, which reaches NetBox and
every switch by cluster DNS. A push runs from the operator's workstation
through port-forwards, so no device API is published outside the cluster.

Nothing is persisted. NetBox and the lab are rebuilt from this repository on
every standup, which is why NetBox's contents are kept as seed files in the
repository. Each switch comes up blank apart from the management and
gNMI configuration containerlab writes; everything else arrives by push.

## How a push works

1. **Plan.** Every switch is rendered from NetBox and compared with its running
   configuration on exactly the paths its template owns. Switches that match
   are left alone; if none differ, nothing is sent.
2. **Commit.** Each differing switch gets one gNMI Set replacing its owned
   paths, all under one commit ID with a 120-second rollback.
3. **Verify.** Each pushed switch must read back exactly as rendered. Then the
   network must behave as NetBox describes, with the expectations derived from
   its cabling and addressing: every link between switches a full OSPF
   adjacency, every loopback reachable from every switch, and the default route
   learned everywhere except where the template originates it. OSPF takes
   seconds to converge, so this polls within the rollback window.
4. **Confirm all, or cancel all.** Anything that fails or interrupts the push
   cancels every committed switch, which rolls back at once; a switch that
   cannot be reached to cancel rolls back when its timer expires.

The pipeline owns specific subtrees of each device: every `ethernet-1/N` port,
`system0`, `network-instance default` and `routing-policy`. `/system` holds the
TLS key and certificate containerlab generated per switch, which NetBox cannot
render, so it stays containerlab's.

A push of some switches is refused while any other switch differs from NetBox,
because the network checks could not pass. `--readback-only` skips the network
checks, for a single-switch canary.

## Proving the rollback

Step 4 is the step that matters when something goes wrong, so it is exercised
on purpose. A proof pushes NetBox's rendered configuration with a fault added
after rendering, and requires every switch to end exactly as it started.

| Scenario             | The fault                                              | What must happen                                             |
| -------------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| Network checks fail  | One link between two switches is disabled              | Every switch commits and reads back, then all are cancelled  |
| A switch rejects     | The last switch pushed is given a value it must refuse | Those before it have already committed, and are cancelled    |

In both, every switch also receives a harmless change, so all four are part of
the transaction. Afterwards a dry run must report zero differences on every
switch and the network checks must pass. The fault is never written to NetBox,
so an interrupted proof leaves the source of truth correct.

## Drift

A dry run is a drift check. Each difference is one of three kinds:

| Kind      | Meaning                                                                     |
| --------- | --------------------------------------------------------------------------- |
| `changed` | The switch has the leaf with a different value from NetBox                  |
| `missing` | NetBox intends the leaf and the switch does not have it                     |
| `extra`   | The switch has a leaf NetBox does not intend, such as a change made by hand |

Read-back uses gNMI's configuration data type, which returns what was
configured and omits SR Linux's defaults, so a clean network reports no
differences.
A push restores whatever a dry run reports.

A watcher runs the same comparison continuously. It holds a gNMI Subscribe
stream to each switch on a single leaf, the switch's own record of when its
configuration last changed, and compares that switch with NetBox a few seconds
after the leaf moves. Every switch is also compared on a timer, which catches
intent that changed in NetBox and was never pushed. Only changes of state are
printed: a switch starts matching, starts differing, differs differently, or
stops answering. The watcher never writes to a switch.

It runs as a single pod in the lab's namespace, reaching the switches and
NetBox by cluster DNS, and is removed with the lab. The same code runs in a
terminal on a workstation, through port-forwards, which is how it was
developed.

Drift can be produced on demand to exercise this. An injector makes one
harmless change on a switch, a description added, altered or removed, so that
each of the three kinds above appears as exactly one difference.

## Layout

| Path                                  | What it holds                                                         |
| ------------------------------------- | --------------------------------------------------------------------- |
| `seed/dcim.yml`                       | Site, device types, roles, platform, devices, loopback interfaces     |
| `seed/cabling.yml`                    | The cabling, and the authority for it                                 |
| `seed/ipam.yml`                       | The addressing plan                                                   |
| `seed/extras.yml`                     | The git data source, and which template serves which role             |
| `templates/srlinux/base.j2`           | Everything every switch shares; renders the owned paths as JSON       |
| `templates/srlinux/<role>.j2`         | One per role, extending the base. Core adds default-route origination |
| `lab/campus.clab.yml`                 | The containerlab topology                                             |
| `lab/topology.cr.yml`                 | The clabernetes custom resource, minus its embedded topology          |
| `lab/watcher.yaml`                    | The drift watcher's Deployment                                        |
| `Dockerfile`                          | The drift watcher's image                                             |
| `netbox/`                             | Values for the NetBox chart, and the PostgreSQL and Valkey it uses    |
| `scripts/lab.sh`, `scripts/netbox.sh` | Lifecycle, and the only operator entry points. Shell only             |
| `src/intentlab/seed.py`               | Validates the seed and loads it into NetBox                           |
| `src/intentlab/push.py`               | Render, diff, push and the commit-confirmed transaction               |
| `src/intentlab/verify.py`             | The network checks, derived from NetBox                               |
| `src/intentlab/render_topology.py`    | Inlines the topology into the custom resource                         |
| `src/intentlab/prove_commit.py`       | The proof that the pygnmi fork's commit confirmed works on SR Linux   |
| `src/intentlab/prove_rollback.py`     | The proof that a failed push leaves every switch as it was            |
| `src/intentlab/inject_drift.py`       | Makes one harmless change on a switch, so there is drift to detect    |
| `src/intentlab/watch.py`              | The drift watcher: a subscription per switch triggering the same diff |
| `docs/ARCHITECTURE.md`                | Why it is shaped this way, decision by decision                       |
| `docs/OPERATIONS.md`                  | Standing the lab up, changing it and tearing it down. Not published   |

## Tests

The test suite needs no lab, no NetBox and no network, and runs in under a
second. It checks that the seed is consistent and its cabling agrees with the
topology in both directions, that broken seeds of each kind are refused before
NetBox is touched, and that the templates render every switch to valid JSON
with exactly the owned paths, passive interfaces only toward the loopback and
hosts, and the default route on the core alone. NetBox's own renders were
compared with these offline renders and parse to identical JSON for every
switch, so the offline stand-ins are a faithful check. The diff is tested for
module prefixes, list order and each kind of difference; the push for
confirming all, cancelling all, and sending nothing when nothing differs; and
the network checks against described healthy and broken states. The faults the
rollback proof injects and each kind of injected drift are applied to the real
renderings and checked for exactly the difference intended, and the watcher is
run against stand-in switches with a clock moved by hand.

## gNMI client

Commit confirmed is a gNMI extension stock pygnmi 0.8.15 cannot send. This
repository depends on a fork, [jpcuckovi/pygnmi](https://github.com/jpcuckovi/pygnmi)
`v0.9.0`, which regenerates its bundled spec for gNMI 0.10.0 and adds the
extension. ARCHITECTURE.md records why a fork, and the proof against SR Linux.

## License

Proprietary. Copyright (c) 2026 NightLabs. All rights reserved. No license is
granted; see [`LICENSE`](LICENSE). Third-party components, including the pygnmi
fork under the BSD 3-Clause License, keep their own terms.
