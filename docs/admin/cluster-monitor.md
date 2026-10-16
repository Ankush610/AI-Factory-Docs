# Cluster monitor

**Interactive Apps → Cluster monitor**, or its card in the **Admin** section of the home page. Only the admin
account sees it. Nobody else can open it, not even by typing its address.

It shows nodes, GPUs, services and alerts on one page. It only reads, never changes anything, and refreshes every
30 seconds. All of its data comes from the cluster's Prometheus on master
([9-monitoring](https://github.com/Ankush610/OOD-Install/blob/main/9-monitoring/README.md)).

## Problems box

At the top of both tabs: every alert that is firing now, the most harmful first (critical, then services, nodes,
GPUs, certificates). Each one has:

- **Show**: opens the node, or the Services tab.
- **How to fix**: opens that alert's row in [Alerts: what to do](https://github.com/Ankush610/OOD-Install/blob/main/9-monitoring/README.md#alerts-what-to-do).

## Nodes tab

- **Top cards:** nodes working, free GPUs by type, CPUs reserved, jobs, GPUs reserved but idle, and services
  answering (click it for the Services tab).
- **Nodes:** cards, or a table with more than 8 nodes. Search by node, user or reason. Filter by type, state, or
  "has a free GPU". Problems come first.
- **One square per GPU:** empty = free, filled = reserved (the colour shows how busy it really is), grey = not
  available. A red ring = faulty (memory errors, needs a reset, slowed down). A dashed red square = missing (Slurm
  expects it, the node doesn't see it).
- **⚠ next to a node:** a faulty or missing GPU, or a core service not running there.
- **Click a node** for its side panel: what needs attention (with How to fix), CPU and memory reserved vs in use,
  each GPU and its health, the last hour, and who runs what there.

## Services tab

- **One sentence first:** "Everything is answering", or what is down and what that breaks (for example, "User
  accounts is down: nobody can log in").
- **What you connect to:** portal, sign-in, LDAP, MLflow, registry, gateway. Each shows OK / Down, since when, the
  response time, and the days left on its certificate (amber under 30, red under 7).
- **Running on the nodes:** the core services on each node, down ones first.
- **Monitoring:** the monitoring's own health (settings loaded, alert rules working, collectors fresh, service
  checks running).

## Links

The tab, the filters and the open node are in the address, so you can bookmark or send a view:
`#tab=services`, `#type=A30&free=1`, `#node=cn02`.

## Alerts

Every alert, what it means and what to do, is in one table:
[Alerts: what to do](https://github.com/Ankush610/OOD-Install/blob/main/9-monitoring/README.md#alerts-what-to-do).
The Cluster monitor's **How to fix** links go there. Change where they point with `RUNBOOK_URL` in `site.conf`.

Email alerts are planned, not built yet.
