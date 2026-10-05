# Architecture

Why this lab is shaped the way it is. Each heading is an assertion; what follows
is the reasoning. Anything not yet decided is listed under a closing "Not yet
decided" heading rather than written up as though it were; at present there is
none.

## The subject is the configuration pipeline, not the network

The point of this repository is the path from intended state to running
configuration: a source of truth holds what the network should be, templates
render each device's configuration from it, a push applies that configuration
with a diff before commit, and a read-back verifies the result and detects
drift. The network exists to give that pipeline something non-trivial to
configure, and is kept as small as will still exercise it.

## NetBox is the source of truth, with its own config templates

NetBox holds the site, the device types and roles, the devices and their
interfaces, the cabling between them, and the addressing. A device's
configuration is rendered from those objects and nothing else. There are no
VLANs to hold, because access is routed. Config contexts are not used either:
the settings they usually carry (NTP, syslog, SNMP, banner) sit under `/system`
on SR Linux, which the pipeline does not write, as the heading on owned subtrees
explains.

Rendering uses NetBox's native config templates (Jinja2), reached through the
device's render-config API endpoint. Templates are kept in this repository and
synced into NetBox rather than edited in its UI, so the repository is the
authority for templates as NetBox is for data.

## Templates are layered, and NetBox syncs them from git

`templates/srlinux/base.j2` holds everything every switch shares, and one
template per role `{% extends %}` it and adds only what that role adds. Each
role template is assigned to its device role in NetBox, which resolves a
device's template from the device, then its role, then its platform. This is
the production shape: one base per platform, thin role layers, so a role gains
behaviour by editing one small file.

Under routed access the roles share their structure, and most differences are
data: addresses come from IPAM, and an interface is a passive OSPF interface
rather than an adjacency when it is the loopback or is cabled to something that
is not a switch, which is how access1's host port becomes the host gateway with
no role logic. The one role behaviour so far is core's: it originates a default
route into OSPF, through a static 0.0.0.0/0 to a blackhole, an ASBR flag and an
export policy that accepts only that route. Aggregation and access extend the
base with nothing added yet, and say where their behaviour would go.

NetBox resolves `{% extends %}` and `{% include %}` only between files of a
data source. Templates stored only in its database cannot find each other,
verified 2026-10-02 (`TemplateNotFound`). So this repository is a git data
source, and `scripts/netbox.sh seed` triggers its sync. The repository is
private: NetBox authenticates with a fine-grained GitHub token limited to
reading this repository's contents, kept in the macOS Keychain and passed in on
each seed, never written to a file. A template edit reaches NetBox once pushed.

**Rejected:** making the repository public to drop the token, and serving the
templates from a ConfigMap mounted into NetBox as a local data source. The
first changes what is private for convenience; the second is not how
production delivers templates.

## The pipeline owns specific subtrees, never the whole device

A template renders a JSON object mapping each owned gNMI path to its intended
value: every `ethernet-1/N` port, `system0`, `network-instance default` and
`routing-policy`. The push replaces exactly those paths and the drift check
reads exactly those paths back, so this object is the single definition of
what the pipeline owns.

A full-device replace is impossible: `/system` holds the TLS profile
containerlab generated per node, with its private key and certificate, which
NetBox cannot render, and a replace without it would cut the gNMI connection it
travels over. `/system`, `mgmt0`, the `mgmt` network instance and `/acl` stay
containerlab's.

All 58 ports are owned, not only the cabled ones. A port with an address in
NetBox is enabled and configured; every other is rendered disabled. So a port
removed from NetBox is disabled on the next push, and a port enabled by hand is
drift. Routed subinterfaces carry an IP MTU of 1400, the largest a clabernetes
link carries without fragmenting, and both ends of every link come from the
same template, so OSPF's MTU check agrees. `max-ecmp-paths` is raised from
SR Linux's default of one, which would otherwise leave access1 using a single
aggregation switch.

**Rejected:** Nautobot with the Golden Config app. It provides intended-config
rendering, backup, compliance and remediation out of the box, which is exactly
the work this repository exists to do by hand. It also brings Nornir and a
larger deployment. NetBox is the more widely deployed of the two, and Nautobot,
as a fork of it, shares the data model, the API shape and the Jinja2
templating, so what is built here carries over. The case for revisiting this is
a specific target environment that runs Nautobot, where hands-on Golden Config
experience is worth more than having built the pipeline by hand.

## The devices are Nokia SR Linux, not Arista cEOS

The sibling `ceos-diag-agent` depends on a cEOS image that is hand-downloaded
from Arista behind an account and has no other source. SR Linux is published
anonymously at `ghcr.io/nokia/srlinux`, is first-class in containerlab, and
exposes JSON-RPC and gNMI for configuration and read-back.

The image is pinned to **26.7.2**, the newest release when checked on
2026-10-01. A lab whose images change silently between runs is not
reproducible.

## The image is pulled from ghcr.io, not mirrored to the lab's registry

The lab's private registry holds images that cannot be obtained again; cEOS is
there for that reason, and netshoot is deliberately not. SR Linux is in the
second category, and a copy in that registry would be a second place the same
image lives with nothing to say which is authoritative.

Size was checked as a reason to mirror anyway and found insufficient: the amd64
image is one layer of 0.75 GB compressed, pulled once per node and cached by
the node's container runtime thereafter. The registry runs on a Raspberry Pi,
so a LAN pull is not clearly faster than a direct one. The remaining argument
for a mirror is availability when ghcr.io or the uplink is down; if that comes
to matter, this decision is revisited.

## Standup follows ceos-diag-agent's pattern, without depending on it

The lab runs under clabernetes on the k3s cluster, as the cEOS lab does. The
containerlab topology file is the only source; the clabernetes custom resource
is generated from it and piped to `kubectl`, never written to disk. The pattern
is copied, not imported: this repository has no code dependency on
`ceos-diag-agent` or `netdiag-core`, and the clabernetes chart pin and its
reasons are inherited from that repository's ARCHITECTURE.md.

**Rejected:** running SR Linux on the workstation under Docker Desktop, despite
the image's native arm64 build. Tried 2026-10-01 with a standalone `docker run`:
`net_inst_mgr` asserted on its first start creating the mgmt network instance's
veth pair ("File exists" for an interface that did not exist), and the device
never came up. containerlab's own macOS guidance runs SR Linux inside a Linux VM
or its devcontainer, neither of which the workstation has. The same image boots
cleanly under clabernetes on the cluster node.

## NetBox runs in the cluster, on the same node as the lab

NetBox is deployed to the same k3s node as the clabernetes topology. The
render-and-push path then reaches each device's management Service by cluster
DNS, and device APIs need not be published outside the cluster.

NetBox has its own namespace, `netbox`, and its own lifecycle in
`scripts/netbox.sh`. `lab.sh down` deletes the lab's namespace, and the lab is
torn down far more often than NetBox, whose first start runs every database
migration.

It is installed from the official chart, `oci://ghcr.io/netbox-community/netbox-chart/netbox`,
pinned to 8.3.90 (NetBox v4.7.1). The chart's bundled PostgreSQL and Valkey
subcharts are disabled: they are Bitnami's, which since 2025 publishes only an
unpinnable `:latest` for free. In their place run the official `postgres:18.6`
and `valkey/valkey:9.1.2` images, the versions the chart's own subcharts ship,
as plain manifests in `netbox/`. None of the three has a persistent volume, as
the next heading explains. The chart generates the superuser password and
secret key itself; only the database password is created by `netbox.sh`, and
nothing secret is committed.

Tooling authenticates with v2 API tokens (`Authorization: Bearer
nbt_<key>.<token>`), provisioned at run time from the superuser's credentials
through `POST /api/users/tokens/provision/`. The chart also stores an
`api_token`, but it is a bare UUID that NetBox 4.7 rejects in either token
format, verified 2026-10-01; v1 tokens are deprecated since 4.6 and removed in
5.0 regardless.

## NetBox is rebuilt from seed files on every standup

The node NetBox runs on cannot be guaranteed to survive; it is reloaded
routinely, and NetBox's database goes with it. NetBox therefore never holds the
only copy of anything. The repository carries seed files describing the site,
device types, roles, devices, interfaces, cables and addressing, along with the
git data source and which template serves which role, and
an idempotent loader applies them through NetBox's API: it creates what is
missing and updates what differs, so a fresh and an already-populated NetBox
converge to the same state.

Standup order is NetBox, then the loader, then render, then push. Recovering
from a reload is running standup again.

A change to intended state is a change to the seed files. An edit made in
NetBox's UI is lost at the next reload unless it is also made there. At
runtime NetBox is still what templates render from and what tooling queries;
the repository is what outlives it.

This differs from a production deployment, where NetBox's database is persisted
and backed up and is edited directly. Moving NetBox to a durable cluster would
restore that model; until then, the seed files are the durable record.

## The topology is core, two aggregation switches and one access switch

```
              core1
             /     \
          agg1     agg2
             \     /
             access1
                |
              host1
```

Three tiers give three device roles, and three roles are what make templating
worth doing: one template per role, with the differences between `agg1` and
`agg2` carried entirely by data.

**Rejected:** reusing `ceos-diag-agent`'s topology. It is a spine-leaf EVPN
fabric with per-switch ASNs and MLAG, which is the data-centre pattern this
design rejects, its dual-homing mechanism does not exist on SR Linux, and it is
nearly twice the size.

## The seed files own the cabling

containerlab needs the links to wire the lab, and NetBox needs the same links as
cables to derive OSPF neighbours, so the cabling exists twice. The seed files
are the authority. The links in `lab/campus.clab.yml` repeat them, and a test
fails if the two disagree in either direction. Generating the topology's links
from the seed instead would remove the copy outright, and replaces the test if
the cabling grows beyond a handful of links.

The switches boot with no startup-config. containerlab enables the gNMI server,
and all other configuration is rendered from NetBox and pushed.

`host1` deletes the default route containerlab gives it through the management
network, which would otherwise make every address reachable and defeat every
data-plane check, as the sibling cEOS lab found. Its address is recorded in
the addressing plan in the seed files, but nothing applies that address or a
route via `access1` to the host yet; that belongs to the data-plane check,
which is not built.

## Access is routed, and the interior runs single-area OSPF

`access1` is the host gateway, and every inter-switch link is a routed
point-to-point link. Both uplinks from `access1` carry OSPF adjacencies, so
traffic toward the core is shared across both aggregation switches by ECMP and
there is no L2 loop for spanning tree to break.

**Rejected:** an L2 trunk from `access1` to both aggregation switches with a
VRRP gateway. It is the more traditional campus design, but it needs spanning
tree, and SR Linux's support for spanning tree is unverified.

OSPF, not BGP, is the interior protocol. A network like this one is a single
autonomous system, and BGP belongs only at its peering points with other
networks. Per-switch ASNs with eBGP between tiers are a data-centre fabric
pattern and would misrepresent the design being modelled.

A single area is the honest design for four switches. The OSPF configuration
is still derived entirely from NetBox: the router ID from the loopback address,
point-to-point interfaces from inter-switch cables and their addressing, and
passive interfaces from host-facing ports. Area membership would become a
custom field if the network ever needed more than one.

The topology has no peering point today, so it runs no BGP. If one is added, it
is a single eBGP session from `core1` to a stand-in upstream network.

## Configuration is pushed and read back over gNMI

gNMI is the transport for push, read-back and verification. It is the
vendor-neutral, model-driven interface, supported across Nokia, Arista, Juniper
and Cisco, where SR Linux's JSON-RPC is specific to one vendor in the way the
cEOS lab's eAPI is.

The unit of change is one SetRequest per device carrying that device's full
rendered configuration, which SR Linux applies atomically. Each SetRequest uses
OpenConfig's commit-confirmed extension, which SR Linux supports at v0.1.0
(per the 24.10 documentation): the change rolls back automatically unless
confirmed, and is confirmed only once verification passes.

gNMI has no on-device diff or validate, which JSON-RPC does. The diff is
computed client-side instead, between the rendered configuration and a gNMI Get
of the running one. That comparison is drift detection, which the pipeline
needs regardless, so the pre-commit diff and the drift check are one mechanism.
Verification after push polls each switch with gNMI Get, every few seconds,
until the OSPF adjacencies and routes that NetBox implies are present or the
rollback window is nearly spent.

Templates render SR Linux's native YANG configuration as JSON, which both
transports accept. The choice of transport is therefore confined to the push
and read-back code; templates and NetBox data are unaffected by it.

**Rejected:** JSON-RPC. It has on-device `diff` and `validate` and is simpler
to call, but it is vendor-specific and has no streaming telemetry.

**Rejected:** NETCONF, despite being the usual Python path to confirmed commit
(ncclient, RFC 6241) and supported by SR Linux. It would make every template
render XML and every drift comparison an XML comparison. Configuration in this
repository is JSON end to end.

gNMI listens on port 57400 with a self-signed certificate in containerlab. The
lab skips certificate verification, and the clabernetes Service publishes
57400.

## The tooling is Python, with pynetbox and a fork of pygnmi

The loader, render driver and push and verify code are Python, matching the
sibling lab repositories. Templating happens server-side in NetBox, so the
client language has no bearing on it. pynetbox, maintained under the
netbox-community organisation, is the NetBox API client. `gnmic` serves for
manual checks against a device.

**Rejected:** Go. The reference gNMI libraries and `gnmic` are Go, and Go would
remove the commit-confirmed gap described below. But that gap sits in the gNMI
code, roughly a third of the repository, while the two largest pieces, the
idempotent seed loader and the JSON drift comparison, are where Go is clumsier
than Python. Go's NetBox client is also less mature than pynetbox.

## The gNMI client is a fork of pygnmi, extended with commit-confirmed

pygnmi as released cannot send the Commit extension the push depends on.
Verified 2026-10-01 against 0.8.15 (released 2025-03-10, still the latest):

- Its bundled `gnmi_ext` protobuf, generated from gNMI v0.8.0, defines only
  RegisteredExtension, MasterArbitration and History. Upstream
  `openconfig/gnmi` defines `Commit` as extension field 4, added in v0.12.0.
- Its `extension` argument converts only `history` and `master_arbitration`
  keys and ignores anything else.
- Its request classes come from that old protobuf, so a Commit message built
  outside pygnmi cannot be attached to a pygnmi SetRequest.

Upstream has had no commit since 2025-03-10, and pull requests opened since
then remain unmerged, so the fix is not expected to land there.

The repository therefore depends on a fork of pygnmi, published under the
repository owner's account and installed as `pygnmi` from a pinned tag. The
fork makes two changes: the bundled gNMI protobufs are regenerated from current
upstream `openconfig/gnmi`, and the extension builder gains a `commit` case
covering commit, confirm, cancel and set-rollback-duration. The work lands on
the fork's own default branch through a pull request there, and is tagged from
it. It is not offered upstream.

What the fork keeps is the reason to fork rather than write a client: TLS
handling including skipped verification, conversion of keyed path strings to
gNMI paths, and Subscribe in all its modes.

**Rejected:** a thin client over stubs generated from upstream protos. It has
no inherited code, but it re-implements TLS, path parsing and Subscribe, which
pygnmi already provides.

**Rejected:** shelling out to `gnmic`. It supports commit-confirmed today, but
puts a Go binary on every host that pushes and turns failures into exit codes
and parsed text.

The regeneration was not purely additive. gNMI removed aliases (the `Alias` and
`AliasList` messages and the alias fields), which pygnmi used; the fork
deprecates them rather than breaking callers. Its version is 0.9.0, a minor bump
following pygnmi's own precedent for a spec rebuild.

The fork was proven on 2026-10-01 against SR Linux 26.7.2 on a single-node
clabernetes topology (`python -m intentlab.prove_commit`): an unconfirmed commit
was applied and then reverted by the device when its rollback duration expired,
and a confirmed commit survived past it. The device reported gNMI 0.10.0, the
specification the fork's protobufs declare.
