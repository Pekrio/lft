# Lightweight Fog Testbed (LFT)

## Description

**LFT** emulates network topologies with Docker containers: each node
(switch, host, controller) is a container, linked to the others with `veth`
pairs. Links get capacity, delay and jitter via `tc netem`. Switches are
**Open vSwitch**, controlled by an **ONOS** instance.

This branch (`chore/reorganize-onos-topologies`) additionally builds ONOS
experiments for **Intent-Based Networking (IBN)** research: a **Deployer**
receives, processes and applies **Nile intents** over the emulated topology,
and an iperf3-based track compares routing modes under network stress
(degradation or link failure):

- **CDN-QoE**: optimal path/server selection from real-time RTT and throughput.
- **LLM**: LLM-based decision-making via an external service.
- **Threshold**: historical `treshold` mode with `supervisor-quantization`.
- **Reactive Forwarding (fwd)**: standard SDN shortest-path routing.

RNP additionally provides a weighted-Dijkstra baseline and distinct
supervisor drift modes. Mode IDs differ between scenarios; see
[`onos_topologies/README.md`](onos_topologies/README.md).

---

## 1. Requirements

- Linux with Docker support (validated on Ubuntu Server 25.04)
- Root/sudo (the CLI and the containers it manages need it)
- `tmux` recommended for persistent runs over SSH

## 2. Installation

One script, one command, from a fresh clone to ready-to-run:

```bash
git clone https://github.com/UnB-COMNET/lft
cd lft
chmod +x dependencies.sh
sudo ./dependencies.sh
```

`dependencies.sh` does everything, in order, and is idempotent (safe to
rerun after a partial/failed run):

1. OS packages: Docker CE + Compose plugin, Open vSwitch, iproute2/iptables,
   Python 3 + venv, firewalld (installed, not enabled), nfdump, git, tmux —
   pinned versions, falls back to latest with a warning if a pin is gone.
2. `.venv` + `pip install -e .` — installs this package with the versions
   pinned in `setup.py`'s `install_requires`.
3. Docker images the ONOS experiments need: pulls `onosproject/onos:2.5.0`,
   builds `alexandremitsurukaihara/lst2.0:openvswitch` and `lft-iperf` from
   `docker/`.

CDN-QoE/LLM/Threshold modes also need externally supplied `deployer` and
`supervisor` images — see [REIN's own setup](https://github.com/UnB-COMNET/REIN)
in `sistemas/REIN` if you're working inside the PIBIC project, or your own
build of those services otherwise. `dependencies.sh` does not build these;
they come from a different repository.

## 3. CLI

After installing, use `sudo lft` to manage topologies interactively.

**Load a topology and open the REPL:**
```bash
sudo lft topology create --preset diamond
# or from a config file:
sudo lft topology create --path onos_topologies/topologies/configs/diamond.py
```

**Start an empty topology manually:**
```bash
sudo lft topology create --manual
```

**REPL commands:**
```
create host <name> <ip>            add a host (iperf3 client)
create server <name> <ip>          add a server (starts iperf3 -s)
create switch <name>               add a switch
connect <name1> <name2>            link two nodes
traffic <ping|iperf> <n1> <n2> [iperf3 flags...]
ls [hosts|switches]
quit / help
```

**Other commands:**
```bash
sudo lft experiment                        # list available experiments
sudo lft experiment <name>                 # run an experiment
sudo lft utils clean                       # remove all Docker containers
```

---

## 4. ONOS experiments and results

The maintained entry points and directory guide are documented in
[onos_topologies/README.md](onos_topologies/README.md). Start with:

```bash
sudo lft experiment diamond --mode fwd --hindering degrade --run-name diamond-first
sudo lft experiment rnp --mode baseline --seed 1 --run-name rnp-first
```

Diamond runs six 60-second measurement windows. RNP runs twelve 60-second
windows with continuous traffic and seeded placement. Both degrade even-numbered
windows; setup and orchestration add time outside the measurement windows.

Results remain under `results/iperf/<run-name>/`. Diamond writes
`iperf_flow_all.csv` and `ping_flow_all.csv`; RNP writes `iperf_all.csv` and
`ping_all.csv`. Both retain OVS outputs. See
[onos_topologies/README.md](onos_topologies/README.md) for modes, external
services, batches and validation requirements.

## 5. Troubleshooting

If you face an issue running any LFT command:

1. Check that `dependencies.sh` ran to completion (`sudo lft` should print
   the banner, not an import error) — rerun it, it's idempotent.
2. Check for leftover containers from a previous run: `docker ps -a`. Remove
   them with `sudo lft utils clean` or `docker rm -f <name>`.
3. Verify the images this experiment needs exist locally (`docker images`) —
   see §2 and, for CDN-QoE/LLM/Threshold, the `deployer`/`supervisor`
   images from REIN.
4. ⚠️ Cleanup routines remove **all** Docker containers on the host. Use a
   dedicated machine, not your daily driver.
