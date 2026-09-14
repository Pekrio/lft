# Lightweight Fog Testbed (LFT) – iPerf Experiment Branch

## Description

This branch extends the **Lightweight Fog Testbed (LFT)** into a research environment for **Intent-Based Networking (IBN)**. It features a **Deployer** that receives, processes and applies **Nile intents** within a virtualized topology built with **ONOS** and **Open vSwitch (OVS)**.

The platform supports a  **iperf3-based experimental track** to compare scenario-specific routing modes under network stress (**degradation** or **link failure**):

- **CDN-QoE**: Algorithm for optimal path and server selection, considering real-time RTT and throughput.
- **LLM**: LLM-based decision-making through the scenario-specific external service.
- **Threshold**: Diamond uses the historical `treshold` mode and `supervisor-quantization`.
- **Reactive Forwarding (fwd)**: Standard SDN shortest-path routing based on hop count.

RNP additionally provides a weighted-Dijkstra baseline and distinct supervisor
drift modes. Mode IDs differ between scenarios; see the package guide.

---

## 1. Requirements

You need:

- (Recommended) Ubuntu Desktop 24.04 LTS
- Docker (with permission to run `sudo docker …`)
- Python 3 and `pip3`
- Git
- `tmux` for persistent runs over SSH

---

## 2. Installation & Image Build

To install the project you need to run:

```
pip3 install profissa_lft
```

Or clone the repository and install locally:

```
git clone https://github.com/alexandrekaihara/lft
cd lft
chmod +x dependencies.sh
sudo ./dependencies.sh
pip3 install -e .
```

## 2.1 Build Docker Images

```
# ONOS 2.5 (Compatible with link-latency app)
sudo docker pull onosproject/onos:2.5.0

# OpenSwitch
cd docker/openswitch && sudo docker build -t alexandremitsurukaihara/lst2.0:openvswitch .

# Iperf client/server
cd docker/iperf && sudo docker build -t lft-iperf .

```
## 3. CLI

After installing, use `sudo lft` to manage topologies interactively.

**Load a topology from config and open the REPL:**
```
sudo lft topology create --path onos_topologies/topologies/configs/diamond.py
```

**Start an empty topology manually:**
```
sudo lft topology create --manual
```

**REPL commands:**
```
create host <name> <ip>       add a host
create switch <name>          add a switch
connect <name1> <name2>       link two nodes
traffic <ping|iperf> <n1> <n2>
ls [hosts|switches]
quit
```

**Other commands:**
```
sudo lft experiment                        list available experiments
sudo lft experiment <name>                 run an experiment
sudo lft utils clean                       remove all Docker containers
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
`ping_all.csv`. Both retain OVS outputs. See the package guide for modes,
external services, batches and validation requirements.

## 5. Troubleshooting
If you face any issue while running any LFT scrips:
1. Check if all dependencies are installed
2. Check if you are using the correct version of Ubuntu Desktop
3. Check if the containers are already instantiated on docker ```docker ps -a```. If so, then remove them by using ```docker system prune``` or forcefully stop them ```docker rm -f containerName```
4. Verify if the docker image that you are trying to instantiate with LFT exists on your local machine ```docker images``` or exists on [Docker Hub|https://hub.docker.com/].
5. Check if the image was built correctly. See docker folder for more information.
